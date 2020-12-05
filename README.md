# BundleSink

Manage multiple webpack entries within an ASPNET Core application.

***

### Requirements

- The library currently supports `netcoreapp3.1` applications.
- You need a webpack configuration for your bundles.

***

## Installation

- From the dotnet CLI:  
`dotnet add package BundleSink`

- Import tag helpers within `_ViewImports.cshtml`:  
`@addTagHelper *, BundleSink`

- From a command line:
`npm install webpack-assets-manifest`

- Update your webpack.config.js adding the plugin:
```js
    plugins: [
        new WebpackAssetsManifest({
            output: path.resolve(__dirname, 'client-manifest.json'),
            entrypoints: true
        })
    ]
```

- Initialize BundleSink in Program.cs:

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder
                .ConfigureBundleSink(builder => {
                    builder.WithWebpack("client-manifest.json", "/dist/");
                })
                .UseStartup<Startup>();
        });
```

> The function `WithWebpack` requires two arguments:
> - The name of the assets manifest created with webpack
> - The **public** output folder (i.e. the folder under wwwroot where the output is being generated by webpack)

***

## Usage

- Place a script-sink in your `_Layout.cshtml` or a razor page's "Scripts" section:  

*e.g.:  

```razor
<h1>Homepage</h1>

@section Scripts {
    <script-sink />
}
```

- Use a webpack entry from within a partial view / view-component, defining **the name of the entry**

```razor
This is a partial view from the homepage

<webpack-entry name="my-homepage-feature" />
```

***

## What happened
The `<webpack-entry>` tag helper marked the specified entry as a dependency.  
The `<script-sink>` tag helper uses the dependencies declared all over the razor pages, taking care of duplicates and required chunks

The html will contain the scripts required for the selected entries, with their hash appended as query string.  
*e.g.*
```html
<script type="text/javascript" src="/dist/my-homepage-feature.js?v=NMaMA8xzap806fSOec7CFpI78hl033lAOIq_Lrr4kmY"></script>
```

***

## Additional options

The webpack-entry accepts more options in form of attributes
- Use the attribute `key` if you need an entry to be imported more than once ... Why would you?  (Their dependencies will be imported once)
- Use the attribute `async` to mark the entry (but not its dependencies) as async
- Use the attribute `defer` to mark the entry (but not its dependencies) as deferred

**Named sinks**
You can also render a webpack-entry to a specific script-sink:
- Use the `name` attribute on the script-sink tag helper
- Use the `sink` attribute on the webpack-entry tag helper

*e.g.*
```razor
<webpack-entry sink="ABOVE" />
<script-sink name="ABOVE />
```

***

## Partial builds

You may want to build one entry at a time, in order to speed up development processes.

The `bundle-sink-webpack-plugin` package provides an option called `partial` which needs to be set to `true` to merge the resulting `client-manifest.json` with the previous one.  

With `partial: true` the clean plugin gets disabled.  

**Automatic configuration**

The automatic configuration is provided by the plugin itself.  
Parses the `env` variable passed to the webpack config and checks whether it refers to an existing entry.

That's how you use it:  
```js
const BundleSinkWebpackPlugin = require('bundle-sink-webpack-plugin');
module.exports = env => {

    let entry = {
        'page-a': './page-a/index.ts',
        'page-b': './page-b/index.ts'
    };

    const bundleSinkWebpackPlugin = new BundleSinkWebpackPlugin({
        clean: true,
        output: path.resolve(__dirname, 'wwwroot/dist/client-manifest.json'),
        entry: entry,
        env: env
    });

    return {
        entry: bundleSinkWebpackPlugin.entry,
        ...ADDITIONAL WEBPACK PARAMETERS,
        plugins: [
            ...bundleSinkWebpackPlugin.plugins
        ]
    };
};
```

You can call webpack this way:  
`webpack --config webpack.config.js --env only=page-a`

**Manual configuration**

You can always **manually** provide the correct `entry` object to webpack and update the bundle-sink options accordingly:  
(*e.g.*)
```js
const BundleSinkWebpackPlugin = require('bundle-sink-webpack-plugin');
module.exports = env => {

    let entry = {
        'page-a': './page-a/index.ts',
        'page-b': './page-b/index.ts'
    }

    const bundleSinkOptions = {
        clean: true,
        output: path.resolve(__dirname, 'wwwroot/dist/client-manifest.json'),
        partial: false
    };
    
    if (env['only']) {
        const selectedEntry = env['only'];
        if (!entry[selectedEntry]) {
            throw new Error(`Syntax: --env only=<entry name>`);
        }
        entry = { [selectedEntry]: entry[selectedEntry] };
        bundleSinkOptions.partial = true;
    }

    const bundleSinkWebpackPlugin = new BundleSinkWebpackPlugin(bundleSinkOptions);

    return {
        entry: entry,
        ...ADDITIONAL WEBPACK PARAMETERS,
        plugins: [
            ...bundleSinkWebpackPlugin.plugins
        ]
    };
};
```

***

## TODO

- Add `requires` and `required-by` attributes for `<webpack-entry />` tag.  
Will help establish priority between entries.  
- Tag helper for generic JS files
- Tag helper for JS excerpts ( literal JS `<script>` tags )
- Add support for CSS
- Add support for partial compilations