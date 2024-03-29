---
layout: post
title:  "Integrating Vue.js with ASP.NET Core MVC"
date:   2019-08-02 13:37:44 +0100
comments: true
---

I'm a big fan of both Vue.js and ASP.NET Core MVC and wanted to see if I could create a tighter integration between the two

When you use the [Vue CLI 3](https://cli.vuejs.org/) (which is awesome), when you compile your project it generates an `index.html` file that contains the references to all the css and javascript for the app

The really nice bit of this is that it generates a hashed path to the file for the purposes of cachebusting. The filepath contains the hash of the file contents, so the user only has to download the file when it's changed

I wanted to get the benefit of this from within my mvc app, so I wanted to see if I could get Vue to generate my `.cshtml` files for me so that as I updated my javascript, my mvc app would be kept up to date

After some poking around under the hood of vue and webpack it turned out it was possible by overriding some settings

One important thing to note here is that I am not creating a `SPA` that is served via MVC. This approach is intended to be for a vue.js app per MVC view

## Create the Vue app via the CLI
Using the CLI, create a new Vue app. I'd suggest creating this in a seperate folder to the MVC app to keep things separated

These are my preferred settings:
#### Settings page 1
1. Babel
1. Typescript
1. Css Pre-processors
1. Linter / Formatter
1. Unit Testing


#### Settings page 2
1. Class-style components
1. Use Babel alongside Typescript
1. Choose node-sass as the pre-processor
1. Choose TSLint as the linter
1. Pick Jest as the unit testing framework

Once this is all done, the CLI will have generated a bunch of files.

## Create a new project
In Visual Studio, create a new `Blank Node.js web application` with `Typescript`

You won't be able to create the project in the same folder the Vue app exists in, so you'll need to:
1. Create the app in a different folder
1. Remove the app in Visual Studio
1. Move the `.nsproj` file into the root of the Vue app
1. In Visual Studio, `add existing project` and select the .nsproj file

## Add the existing files from the Vue app
In the node app, include the existing `src` and `test` folders

I also added all the supporting files (eg. package.json, tslint.json etc. to make my life easier)

Now you can delete the `public` folder as we're about to make that obsolete

## Configuration
In the root of the node project, add a new `vue.config.js` file

This will be read and merged into the webpack config at build time, so while it looks like the webpack.config you might be used to, there are some subtle differences if you want to do more advanced stuff with it

In this file, add the following:

{% highlight javascript %}
const path = require('path');
const del = require('delete');

const webapp = path.resolve(__dirname, '../Mvc.App');
const wwwroot = `${webapp}/wwwroot`;

del(`${wwwroot}/*.hot-update.*`, { force: true },
    function (err, deleted) {
        if (err) throw err;
        console.log(deleted);
    });

const minify = {
    caseSensitive: true,
    removeComments: false,
    collapseWhitespace: false,
    removeRedundantAttributes: false,
    collapseBooleanAttributes: false,
    removeScriptTypeAttributes: false,
    includeAutoGeneratedTags: false,
    keepClosingSlash: true,
};


var pages = {};
[
    { name: 'home', app: 'home/home.ts', view: 'Home/Home.cshtml' },
].forEach(function (entry) {
    const page = {
        entry: `src/pages/${entry.app}`,
        template: 'src/template.cshtml',
        filename: `${webapp}/Views/${entry.view}`,
        minify: minify,
        inject: false
    };

    pages[entry.name] = page;
});

module.exports = {
    pages: pages,
    outputDir: wwwroot,
    productionSourceMap: false,
};
{% endhighlight %}
&nbsp;

So what does this all do?

Starting at the top, we setup the relative paths to both the root of the MVC app, and the wwwroot folder

If we run this using `watch` then we will generate lots of hot-update javascript files. We can't use the `clean` argument as that will wipe out the entire wwwroot folder, which would wipe out any other files the MVC app may contain, so instead we run the del command and target these files specifically

The `minifyOptions` affect how the html that is output is generated. Because we will generate cshtml, not plain html, we need to tweak some of these so our final output isn't a garbled mess

Next we need to generate our `pages` object, which contains all of our mappings between vue apps, template files the the resultant MVC views. We generate this by iterating over an array of objects, which contains the name, path to the vue app and path to the destination cshtml file and then outputting our required options

Finally, we pass our options to module.exports where it will be merged with other webpack options

## The cshtml template
Now we have our configuration, we need need to create the template file that we'll use to generate the chstml files

In the src folder of the Node app, create an `template.cshtml` file and add the following code to it

{% highlight html %}
@section head
{
    <% for(var i=0; i < htmlWebpackPlugin.files.js.length; i++) { %>
        <vue-link href="<%= htmlWebpackPlugin.files.js[i] %>" rel="preload" as="script" />
    <% } %>
    <% for(var i=0; i < htmlWebpackPlugin.files.css.length; i++) { %>
        <vue-link href="<%= htmlWebpackPlugin.files.css[i] %>" rel="preload" as="style" />
        <vue-link href="<%= htmlWebpackPlugin.files.css[i] %>" rel="stylesheet" />
    <% } %>
}

@section scripts
{
    <% for(var i=0; i < htmlWebpackPlugin.files.js.length; i++) { %>
        <vue-script type="text/javascript" src="<%= htmlWebpackPlugin.files.js[i] %>"></vue-script>
    <% } %>
}
<div id="app"></div>

{% endhighlight %}
&nbsp;

This code assumes you will have `head` and `scripts` sections in the `_layout` file for the MVC app

It will output the tags for both the css and javascript the app generates and will also generate `preload` tags for them

The sharp eyed reader will have noticed that this outputs `vue-link` and `vue-script` tags. Unfortunately, the paths that are written out will contain `wwwroot`, meaning that they will result in 404 errors at runtime. To get around this, we output custom tags that we can then target in the MVC app to remove the extra path values

## MVC Tag helpers
Now we need to fix the extra wwwroot in the paths, and to do this, we'll use the `TagHelper` class.

In the MVC app, add the following code
{% highlight csharp %}
[HtmlTargetElement("vue-link")]
public class VueLinkTagHelper : TagHelper
{
    [ViewContext]
    public ViewContext ViewContext { get; set; }

    private readonly IUrlHelperFactory _urlHelperFactory;

    public VueLinkTagHelper(IUrlHelperFactory urlHelperFactory)
    {
        _urlHelperFactory = urlHelperFactory;
    }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        var helper = _urlHelperFactory.GetUrlHelper(ViewContext);
        var href = output.Attributes["href"].Value.ToString().Replace("/wwwroot/", "~/");
        output.Attributes.SetAttribute("href", helper.Content(href));
        output.TagName = "link";
    }
}
{% endhighlight %}
&nbsp;

{% highlight csharp %}
[HtmlTargetElement("vue-script")]
public class VueScriptTagHelper : TagHelper
{
    [ViewContext]
    public ViewContext ViewContext { get; set; }

    private readonly IUrlHelperFactory _urlHelperFactory;

    public VueScriptTagHelper(IUrlHelperFactory urlHelperFactory)
    {
        _urlHelperFactory = urlHelperFactory;
    }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        var helper = _urlHelperFactory.GetUrlHelper(ViewContext);
        var src = output.Attributes["src"].Value.ToString().Replace("/wwwroot/", "~/");
        output.Attributes.SetAttribute("src", helper.Content(src));
        output.TagName = "script";
    }
}
{% endhighlight %}
&nbsp;

Finally, reference the helpers in the `_ViewImports.cshtml` file

```
@addTagHelper *, Namespace.The.TagHelpers.Are.In
```
&nbsp;

Now, when the MVC app generates the html to send to the client, the taghelpers will run and replace the errant path information for us

## Update the package.json scripts
Finally, we want to tweak our `package.json` file to have the following scripts
{% highlight json %}
  "scripts": {
    "build": "vue-cli-service build --no-clean --mode development",
    "build-watch": "vue-cli-service build --no-clean --watch --mode development",
    "build-prod": "vue-cli-service build --no-clean",
    "lint": "vue-cli-service lint",
    "test:unit": "vue-cli-service test:unit"
  },
{% endhighlight %}
&nbsp;

Mainly what we are doing here is adding the `--no-clean` flag to the commands


## Summary
So, with all of this in place, you now have the ability to generate your views directly from a Vue app

Should you do this? I don't know!

It works for me and what i'm trying to achieve. It's not as clean as i'd like and there's a bit more setup than is ideal, but it was interesting to figure out how to make it happen
