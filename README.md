A fast, simple static site builder, inspired by [Middleman](https://middlemanapp.com/). Built for news.

## Why another static site generator?

We've been using Middleman in Vox Media newsrooms for many years to build static articles and embeddable interactives. But as we've built larger projects and projects that require frequent updates, we've found that Middleman's extensive dependencies and complexity can cause high resource use and long build times. However Middleman is great for a lot of use-cases and you should use it if it does what you need.

We need something simple and fast. Something that provides only the functionality we need and uses a few simple dependencies.

This is a simple ruby library and doesn't include any setup or project creation tools. If you want a helpful setup wizard or an ecosystem of extensions and tooling, checkout Middleman or a different static site generator.

## Quick Start

Viz Builder is a simple library that leaves it to you to load and invoke it. There is no shell command for running it or setting up a new project.

To your project's `Gemfile`, add:

```ruby
gem 'vizbuilder'
```

Then in a new or existing file, you will `require` it, configure it and either run as a development server or to create static html and assets ready for upload.

Here is an example file `app.rb` that will load, configure and run Viz Builder:

```ruby
require 'vizbuilder'

app = VizBuilder.new do
  set :some_global_config_thing, 'hello world'
  add_page 'index.html', template: 'index.html.erb'
end

# if run with the argument `runserver` run as a development server
if ARGV[0] == 'runserver'
  app.runserver!
else
  app.build!
end
```

And then execute your app:

```
$ bundle exec ruby app.rb runserver
```

## Data

Viz Builder manages configuration, sitemap and project data in three hashes that you can access directly in templates or the configuration block.

Configuration variables added via the `set` method and can be accessed in the `config` Hash:

```ruby
set :foo, 'bar'
config[:foo] == 'bar'
```

Data variables can be added via the `add_data` method and can be accessed in the `data` Hash:

```ruby
add_data :scraped, foo: 'bar'
data[:scraped] == { foo: 'bar' }
# data[:scraped][:foo]
#   or
# data.dig(:scraped, :foo)
```

Viz Builder automatically loads all JSON and YAML files located in the `data` directory. The base file name will be used as the name:

```ruby
# assuming a file called `data/my_content.json`
data[:my_content] == JSON.parse(File.read('data/my_content.json'))
```

Viz Builder keeps track of pages in a `sitemap` Hash. Pages are added with `add_page`:

```ruby
add_page 'index.html', template: 'index.html.erb', foo: 'bar'
# sitemap['index.html'] == { template: 'index.html.erb', foo: 'bar' }
```

In the context of the template for this page, the data given to `add_page` is accessible in the `page` hash.

In `index.html.erb`:
```ERB
<% page == { template: 'index.html.erb', foo: 'bar' } %>
<%=page[:foo] # 'bar' %>
```

## Configuration

A new `VizBuilder` instance is configured via a block passed into the constructor:

```ruby
app = VizBuilder.new do
  set :some_global_config_thing, 'hello world'
  add_page 'index.html', template: 'index.html.erb'
end
```

You can add or adjust configuration directly on the `VizBuilder#config` instance, but
it may not work as expected:

```ruby
app.config.set :some_global_config_thing, 'a different thing'
app.config.add_page 'index.html', template: 'index.html.erb'
```

There are a few config settings used by Viz Builder:

#### http_prefix

Used in all modes for setting canonical urls and links between pages of the site. Used by the `canonical_url` helper detailed below. Defaults to `'/'`.

#### asset_http_prefix

Used in all modes for setting asset urls. Used by the `asset_path` helper detailed below. Defaults to `http_prefix` or `'/'`.

#### layout

An optional setting that will render all pages inside the specified ERB layout file.

## Templates

Templates are loaded by path from the root of your project directory. There are currently no special locations from which template files are loaded.  If you keep your templates in a subdirectory of your project, you'll need to include the directory when specifying the file:

```ruby
# keeping our templates in a subdirectory called 'templates'
app = VizBuilder.new do
  add_page 'index.html', template: 'templates/index.html.erb'
  add_page 'article.html', template: 'templates/article.html.erb'
end
```

#### Layouts

You can have your templates render inside a `layout` file by either setting a global `:layout` or by setting the layout on a page-by-page basis.

To render all pages in the app in a layout:

```ruby
app = VizBuilder.new do
  set :layout, 'layout.html.erb'
  add_page 'index.html', template: 'index.html.erb'
  add_page 'article.html', template: 'article.html.erb'
end
```

If you don't want a layout used for all pages, or if one page needs to use a different layout, you can pass `layout` to `add_page`:

```ruby
app = VizBuilder.new do
  add_page 'index.html', template: 'index.html.erb', layout: 'layout.html.erb'
  add_page 'article.html', template: 'article.html.erb', layout: 'article_layout.html.erb'
  add_page 'snippet.html', template: 'snippet.html.erb' # no layout is used for this one
end
```

## Helpers

There are a few helpers included in Viz Builder, and you can add your own as well. Helpers are methods that are usable from a template or from the configuration block.

To add helpers:

```ruby
app = VizBuilder.new do
  ...

  helpers do
    def do_something(args)
      # do stuff
      'stuff is done'
    end
  end
end
```

You can also break your helpers out into a separate file and module.

So in a new file called `helpers.rb` we add:

```ruby
module MyHelpers
  def do_something(args)
    # do stuff
    'stuff is done'
  end
end
```

Then in your `app.rb` add:

```ruby
require 'helpers.rb'

app = VizBuilder.new do
  ...

  helpers MyHelpers
end
```

Helpers can be used immediately in the configuration block in addition to in the template:

```ruby
app = VizBuilder.new do
  ...

  helpers do
    def do_something(args)
      # do stuff
      'stuff is done'
    end
  end

  do_something :stuff
end
```

WARNING: You should probably not use instance variables in helpers (`@myvariable`), because the instance variables are not shared between the configuration block and templates. You can use `set` and `add_data` to stash data between helper uses.

### Built-in helpers

#### render

Render a template or partial and return the output. Takes the path to the template and a hash of local variables.

```ruby
<%=render 'list.html.erb', list_items: items %>
```

#### include_file

Load and return the contents of a file. Good for inlining CSS or SVG content.

```erb
<%=include_file 'logo.svg' %>
```

#### http_prefix

Returns the value of the config item `http_prefix`. Should represent the root path to the site.

#### asset_http_prefix

Returns the value of the config item `asset_http_prefix` or `http_prefix`. Should represent the root path to all assets for the site.

#### asset_path

Returns a working url pointing to a given asset. Any provided arguments will be added to the returned url:

```erb
<%=asset_path :images, 'logo.png' %>
<!-- outputs {asset_http_prefix}/images/logo.png --->
<%=asset_path 'images/logo.png' %>
<!-- also outputs {asset_http_prefix}/images/logo.png --->
```

#### canonical_url

Returns the full domain name and path to the root of the site. Any provided arguments will be added to the returned url:

```erb
<%=canonical_url 'page1' %>
<!-- outputs {http_prefix}/page1 --->
```

#### production?

True if Viz Builder is running in production mode. Necessary for setting different http prefix settings for production vs development.

```ruby
app = VizBuilder.new do
  ...
  if production?
    set :http_prefix, 'http://www.example.com'
    set :asset_http_prefix, 'http://cdn.example.com'
  end
  ...
end
```

#### development?

True if Viz Builder is running in development mode. Opposite of `production?`.

#### build?

True if Viz Builder is building out all the pages and not running as a rack app (aka web server). Usually synonymous with `production?`.

#### server?

True if Viz Builder is running as a rack app (aka web server). Usually synonymous with `development?`.

## Assets

You can generate Javascript and CSS using the same template processing used for HTML files. Just add a JS or CSS file as a page in your app configuration:

```ruby
app = VizBuilder.new do
  add_page 'index.html', template: 'index.html.erb'
  add_page 'app.js', template: 'app.js.erb'
  add_page 'app.css', template: 'app.css.erb'
end
```

Viz Builder will also look in the `prebuild` directory for static assets. The builder will copy these files into the `build` directory during the build process, or it will serve these files directly during development server.

Viz Builder will not transform or minify assets for you. It will not handle SCSS and it will not compile ES6+ down to ES5 Javascript. I recommend using something like [webpack](https://webpack.js.org/) to handle processing javascript and stylesheets. Whatever tool you use should be configured to save output files to `prebuild`.

If you only have a little script and style in your site, we recommend just using vanilla Javascript and CSS.

#### Asset hashing

Viz Builder automatically adds sha1 crypto hashes to the filenames of certain asset files when building out a site. Any files from the `prebuild` directory will get these hashes. You can have these crypto hashes added to any page with the `digest` option:

```ruby
app = VizBuilder.new do
  add_page 'app.js', template: 'app.js.erb', digest: true
end
```

WARNING: You probably don't want to use the `digest` on HTML pages.

You must use the `asset_path` helper to always get a proper URL for your asset files:

```ERB
<script src="<%=asset_path 'javascript/app.js' %>"></script>
<link ref="stylesheet" href="<%=asset_path 'stylesheets/app.css' %>" />
```

WARNING: If you do not use the `asset_path` helper, stuff will work in development but will break when you deploy.

## Contributing

Fork this repo, create a new branch on your fork, and make your changes there.
Open a pull request on this repo for consideration.

If its a small bugfix, feel free making the changes and opening a PR. If it's a
feature addition or a more substantial change, please open a github issue
outlining the feature or change. This is just to save you time and make sure
your efforts can get aligned with other folks' plans.
