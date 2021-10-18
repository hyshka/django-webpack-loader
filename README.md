# django-webpack-loader

[![Build Status](https://circleci.com/gh/django-webpack/django-webpack-loader/tree/master.svg?style=svg)](https://circleci.com/gh/django-webpack/django-webpack-loader/tree/master)
[![Coverage Status](https://coveralls.io/repos/github/django-webpack/django-webpack-loader/badge.svg?branch=master)](https://coveralls.io/github/django-webpack/django-webpack-loader?branch=master)
![pyversions](https://img.shields.io/pypi/pyversions/django-webpack-loader)
![djversions](https://img.shields.io/pypi/djversions/django-webpack-loader)


Use webpack to generate your static bundles without django's staticfiles or opaque wrappers.


Django webpack loader consumes the output generated by [webpack-bundle-tracker](https://github.com/owais/webpack-bundle-tracker) and lets you use the generated bundles in django.

A [changelog](CHANGELOG.md) is also available.


## Compatibility

Test cases cover Django>=2.0 on Python>=3.5. 100% code coverage is the target so we can be sure everything works anytime. It should probably work on older version of django as well but the package does not ship any test cases for them.


## Install

```bash
npm install --save-dev webpack-bundle-tracker

pip install django-webpack-loader
```

## Configuration

### Configuring `webpack-bundle-tracker`
Before configuring `django-webpack-loader`, let's first configure what's necessary on `webpack-bundle-tracker` side. Update your Webpack configuration file (it's usually on `webpack.config.js` in the project root). Make sure your file looks like this (adapt to your needs):

```javascript
const path = require('path');
const webpack = require('webpack');
const BundleTracker = require('webpack-bundle-tracker');

module.exports = {
  context: __dirname,
  entry: './assets/js/index',
  output: {
    path: path.resolve('./assets/webpack_bundles/'),
    filename: "[name]-[hash].js"
  },
  plugins: [
    new BundleTracker({filename: './webpack-stats.json'})
  ],
}
```

The configuration above expects the `index.js` (the app entrypoint file) to live inside the `/assets/js/` directory (this guide going forward will assume that all front-end related files are placed inside the `/assets/` directory, with the different kinds of files arranged within its subdirectories).

The generated compiled files will be placed inside the `/assets/webpack_bundles/` directory and the file with the information regarding the bundles and assets (`webpack-stats.json`) will be stored in the project root.

### Compiling the front-end assets

You must generate the front-end bundle using `webpack-bundle-tracker` before using `django-webpack-loader`. You can compile the assets and generate the bundles by running: 

```bash
npx webpack --config webpack.config.js --watch
```

This will also generate the stats file. You can also refer to how `django-react-boilerplate` configure the [package.json](https://github.com/vintasoftware/django-react-boilerplate/blob/master/package.json) scripts for different situations.

> ⚠️ Hot reload is available through a specific config. Check [this section](#hot-reload).

> ⚠️ This is the recommended usage for the development environment. For **usage in production**, please refer to [this section](#usage-in-production)

### Configuring the settings file
First of all, add `webpack_loader` to `INSTALLED_APPS`.
```python
INSTALLED_APPS = (
  ...
  'webpack_loader',
  ...
)
```

Below is the recommended setup for the Django settings file when using `django-webpack-loader`.

```python
WEBPACK_LOADER = {
  'DEFAULT': {
    'CACHE': not DEBUG,
    'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats.json'),
    'POLL_INTERVAL': 0.1,
    'IGNORE': [r'.+\.hot-update.js', r'.+\.map'],
  }
}
```

For that setup, we're using the `DEBUG` variable provided by Django. Since in a production environment (`DEBUG = False`) the assets files won't constantly change, we can safely cache the results (`CACHE=True`) and optimize our flow, as `django-webpack-loader` will read the stats file only once and store the assets files paths in memory. From that point pnwards, it will use these stored paths as the source of truth. If `CACHE=False`, we'll always read the stats file to get the assets paths.
> ⚠️ If `CACHE=True`, any changes made in the assets files will only be read when the web workers are restarted.

During development, when the stats file changes a lot, we want to always poll for its updated version (in our case, we'll fetch it every 0.1s, as defined on `POLL_INTERVAL`).
> ⚠️ In production (`DEBUG=False`), we'll only fetch the stats file once, so `POLL_INTERVAL` is ignored.

While `CACHE` isn't directly related to `POLL_INTERVAL`, it's interesting to keep `CACHE` binded to the `DEBUG` logic value (in this case, the negation of the logic value) in order to only cache the assets in production, as we'd not continuously poll the stats file in that environment.

The `STATS_FILE` parameter represents the output file produced by `webpack-bundle-tracker`. Since in the Webpack configuration file we've named it `webpack-stats.json` and stored it on the project root, we must replicate that setting on the back-end side.

`IGNORE` is a list of regular expressions. If a file generated by Webpack matches one of the expressions, the file will not be included in the template.

### Extra settings

- `TIMEOUT` is the number of seconds webpack_loader should wait for webpack to finish compiling before raising an exception. `0`, `None` or leaving the value out of settings disables timeouts

- `LOADER_CLASS` is the fully qualified name of a python class as a string that holds the custom webpack loader. This is where behavior can be customized as to how the stats file is loaded. Examples include loading the stats file from a database, cache, external url, etc. For convenience, `webpack_loader.loader.WebpackLoader` can be extended. The `load_assets` method is likely where custom behavior will be added. This should return the stats file as an object.

  Here's a simple example of loading from an external url:

  ```py
  import requests
  from webpack_loader.loader import WebpackLoader

  class ExternalWebpackLoader(WebpackLoader):
    def load_assets(self):
      url = self.config['STATS_URL']
      return requests.get(url).json()
  ```

## Rendering
In order to render the front-end code into the Django templates, we use the `render_bundle` template tag.

Its behavior is to accept a string with the name of an entrypoint from the stats file (in our case, we're using `main`) and it'll proceed to include all files under that entrypoint. You can read more about the entrypoints concept [here](https://webpack.js.org/concepts/entry-points/).

> ⚠️ You can also check an example on how to use multiple `entry` values [here](https://github.com/django-webpack/django-webpack-loader/tree/master/examples/code-splitting).

Below is the basic usage for `render_bundle` within a template.

```HTML+Django
{% load render_bundle from webpack_loader %}

{% render_bundle 'main' %}
```

That will render the proper `<script>` and `<link>` tags needed in your template.

## Running in development
For `django-webpack-loader` to work, you must run the webpack pipeline. Please refer to [this section](#compiling-the-front-end-assets).

In summary, you should do the following:

```bash
# in one shell
npx webpack --config webpack.config.js --watch

# in another shell
python manage.py runserver
```

> ⚠️ You can also check [this example](https://github.com/django-webpack/django-webpack-loader/tree/master/examples/simple) on how to run a project with `django-webpack-loader` and `webpack-bundle-track`.

## Usage in production
We recommend that you keep your local bundles and the stats file outside the version control, having a production pipeline that will compile and collect the assets during the deployment phase.

You must add `STATICFILES_DIRS` to your settings file, pointing to the directory where the static files are located. This will let `collectstatic` know where it should look at:
```python
STATICFILES_DIRS = (
  os.path.join(BASE_DIR, 'assets'),
)
```

Below are the commands that should be run to compile and collect the static files (please note this may change from platform to platform):
```
npm run build
python manage.py collectstatic --noinput
```

First we build the assets and, since we have `webpack-bundle-tracker` in our front-end building pipeline, the stats file will be populated. Then, we manually run collecstatic to collect the compiled assets.

> ⚠️ Heroku is one plataform that automatically runs collectstatic for you, so you need to set `DISABLE_COLLECTSTATIC=1` environment var. Instead, you must manually run collectstatic after running webpack. In Heroku, this is achieved with a `post_compile` hook. You can see an example on how to implement this flow on [django-react-boilerplate](https://github.com/vintasoftware/django-react-boilerplate/tree/master/bin).

However, production usage for this package is **fairly flexible**. Other approaches may include keeping the production bundles in the version control and take that responsibility from the automatic pipeline. However, you must remember to always build the frontend and generate the bundle before pushing to remote.

## Usage in tests
There are 2 approaches for when `render_bundle` shows up in tests, since we don't have `webpack-bundle-tracker` at that point to generate the stats file.

1. The first approach is to have specific settings for them (which is how we approach on our [tests](https://github.com/django-webpack/django-webpack-loader/blob/master/tests/app/settings.py#L111-L125)), such as done [here](https://github.com/django-webpack/django-webpack-loader/issues/187#issuecomment-470055769). Please note that it's necessary to have a pre-made stats file for the tests (which in general can be empty, such as [here](https://github.com/django-webpack/django-webpack-loader/issues/187#issuecomment-464250721)).

2. The second approach is to leverage [`LOADER_CLASS` overriding](#extra-settings) for the test settings and customize the `get_bundle` method to return the url of a stats file. Note that, using this approach, the stats file doesn't have to [exist](https://github.com/django-webpack/django-webpack-loader/issues/187#issuecomment-901449290).

## Advanced Usage
### Rendering by file extension

`render_bundle` also takes a second argument which can be a file extension to match. This is useful when you want to render different types for files in separately. For example, to render CSS in head and JS at bottom we can do something like this,

```HTML+Django
{% load render_bundle from webpack_loader %}

<html>
  <head>
    {% render_bundle 'main' 'css' %}
  </head>
  <body>
    ....
    {% render_bundle 'main' 'js' %}
  </body>
</head>
```

### Using preload
The `is_preload=True` option in the `render_bundle` template tag can be used to add `rel="preload"` link tags.

```HTML+Django
{% load render_bundle from webpack_loader %}

<html>
  <head>
    {% render_bundle 'main' 'css' is_preload=True %}
    {% render_bundle 'main' 'js' is_preload=True %}

    {% render_bundle 'main' 'css' %}
  </head>

  <body>
    {% render_bundle 'main' 'js' %}
  </body>
</html>
```

### Accessing other webpack assets
`webpack_static` template tag provides facilities to load static assets managed by webpack in Django templates. It is like Django's built in `static` tag but for webpack assets instead.

In the below example, `logo.png` can be any static asset shipped with any npm package.

```HTML+Django
{% load webpack_static from webpack_loader %}

<!-- render full public path of logo.png -->
<img src="{% webpack_static 'logo.png' %}"/>
```
The public path is based on `webpack.config.js` [output.publicPath](https://webpack.js.org/configuration/output/#output-publicpath).

Please note that this approach will use the original asset file, and not a post-processed one from the Webpack pipeline, in case that file had gone through such flow (i.e.: You've imported an image on the React side and used it there, the file used within the React components will probably have a hash string on its name, etc. This processed file will be different than the one you'll grab with `webpack_static`).

### Use `skip_common_chunks` on `render_bundle`
You can use the parameter `skip_common_chunks=True` to specify that you don't want an already generated chunk be generated again in the same page.

In order for this option to work, `django-webpack-loader` requires the `request` object to be in the context, to be able to keep track of the generated chunks.

The `request` object is passed by default via the `django.template.context_processors.request` middleware with using the Django built-in templating system, and also with using Jinja2.

If you don't have `request` in the context for some reason (e.g. using `Template.render` or `render_to_string` directly without passing the request), you'll get warnings on the console.

### Appending file extensions
The `suffix` option can be used to append a string at the end of the file URL. For instance, it can be used if your webpack configuration emits compressed `.gz` files.
qwe
```HTML+Django
{% load render_bundle from webpack_loader %}
<html>
  <head>
    <meta charset="UTF-8">
    <title>Example</title>
    {% render_bundle 'main' 'css' %}
  </head>
  <body>
    {% render_bundle 'main' 'js' suffix='.gz' %}
  </body>
</html>
```

### Multiple Webpack configurations
Version 1.0 and up of `django-webpack-loader` also supports multiple Webpack configurations. The following configuration defines 2 Webpack stats files in settings and uses the `config` argument in the template tags to influence which stats file to load the bundles from.

```python
WEBPACK_LOADER = {
    'DEFAULT': {
        'BUNDLE_DIR_NAME': 'bundles/',
        'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats.json'),
    },
    'DASHBOARD': {
        'BUNDLE_DIR_NAME': 'dashboard_bundles/',
        'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats-dashboard.json'),
    }
}
```

```HTML+Django
{% load render_bundle from webpack_loader %}

<html>
  <body>
    ....
    {% render_bundle 'main' 'js' 'DEFAULT' %}
    {% render_bundle 'main' 'js' 'DASHBOARD' %}

    <!-- or render all files from a bundle -->
    {% render_bundle 'main' config='DASHBOARD' %}

    <!-- the following tags do the same thing -->
    {% render_bundle 'main' 'css' 'DASHBOARD' %}
    {% render_bundle 'main' extension='css' config='DASHBOARD' %}
    {% render_bundle 'main' config='DASHBOARD' extension='css' %}

    <!-- add some extra attributes to the tag -->
    {% render_bundle 'main' 'js' 'DEFAULT' attrs='async charset="UTF-8"'%}
  </body>
</head>
```

### File URLs instead of html tags

If you need the URL to an asset without the HTML tags, the `get_files` template tag can be used. A common use case is specifying the URL to a custom css file for a Javascript plugin.

`get_files` works exactly like `render_bundle` except it returns a list of matching files and lets you assign the list to a custom template variable. For example:

```HTML+Django
{% get_files 'editor' 'css' as editor_css_files %}
CKEDITOR.config.contentsCss = '{{ editor_css_files.0.publicPath }}';

<!-- or list down name, path and download url for every file -->
<ul>
{% for css_file in editor_css_files %}
    <li>{{ css_file.name }} : {{ css_file.path }} : {{ css_file.publicPath }}</li>
{% endfor %}
</ul>
```

### Code splitting
In case you wish to use [code-splitting](https://webpack.js.org/guides/code-splitting/), follow the recipe below on the Javascript side.

Create your entrypoint file and add elements to the DOM, while leveraging the lazy imports.
```js
// src/principal.js
function getComponent() {
  return import(/* webpackChunkName: "lodash" */ 'lodash').then(({ default: _ }) => {
    const element = document.createElement('div');

    element.innerHTML = _.join(['Hello', 'webpack'], ' ');

    return element;
  }).catch(error => 'An error occurred while loading the component');
}

getComponent().then((component) => {
  document.body.appendChild(component);
})
```

On your configuration file, do the following:
```js
// webpack.config.js
module.exports = {
  context: __dirname,
  entry: {
    principal: './src/principal',
  },
  output: {
    path: path.resolve('./dist/'),
    // publicPath should match your STATIC_URL config.
    // This is required otherwise webpack will try to fetch 
    // our chunk generated by the dynamic import from "/" instead of "/dist/".
    publicPath: '/dist/', 
    chunkFilename: '[name].bundle.js',
    filename: "[name]-[hash].js"
  },
  plugins: [
    new BundleTracker({ filename: './webpack-stats.json' })
  ]
}
```

If you're using Webpack 5 instead of 4, do the following:
- Change `filename: "[name]-[hash].js"` to `filename: "[name]-[fullhash].js"`;
- Remove `/* webpackChunkName: "lodash" */`, which is not needed anymore.

On your template, render the bundle as usual:

```HTML+Django
<!-- index.html -->
{% load render_bundle from webpack_loader %}

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My test page</title>
  </head>
  <body>
    <p>This is my page</p>

    {% render_bundle 'principal' 'js' %}
  </body>
</html>
```

### Hot reload
In case you wish to enable hot reload for your project using `django-webpack-loader` and `webpack-bundle-tracker`, please check out [this example](https://github.com/django-webpack/django-webpack-loader/tree/master/examples/hot-reload), in particular how [server.js](https://github.com/django-webpack/django-webpack-loader/blob/master/examples/hot-reload/server.js) and [webpack.config.js](https://github.com/django-webpack/django-webpack-loader/blob/master/examples/hot-reload/webpack.config.js) are configured.

### Jinja2 Configuration

If you need to output your assets in a jinja template, we provide a Jinja2 extension that's compatible with the [Django Jinja](https://github.com/niwinz/django-jinja) module and Django 1.8.

To install the extension add it to the django_jinja `TEMPLATES` configuration in the `["OPTIONS"]["extension"]` list.

```python
from django_jinja.builtins import DEFAULT_EXTENSIONS
TEMPLATES = [
  {
    "BACKEND": "django_jinja.backend.Jinja2",
    "OPTIONS": {
      "extensions": DEFAULT_EXTENSIONS + [
        "webpack_loader.contrib.jinja2ext.WebpackExtension",
      ],
    }
  }
]
```

Then in your base jinja template:

```HTML
{{ render_bundle('main') }}
```

## Migrating from version < 1.0.0

In order to use `django-webpack-loader>=1.0.0`, you must ensure that `webpack-bundle-tracker@1.0.0` is being used on the JavaScript side. It's recommended that you always keep at least minor version parity across both packages, for full compatibility.

This is necessary because the formatting of `webpack-stats.json` that `webpack-bundle-tracker` outputs has changed starting at version `1.0.0-alpha.1`. Starting at `django-webpack-loader==1.0.0`, this is the only formatting accepted here, meaning that other versions of that package don't output compatible files anymore, thereby breaking compatibility with older `webpack-bundle-tracker` releases.
