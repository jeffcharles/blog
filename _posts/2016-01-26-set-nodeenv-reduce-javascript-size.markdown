---
title: Set NODE_ENV to reduce your JavaScript size
---

With Redux's 3.0.6 release, I learned about a new way to shrink the size of some of my webpages. A number of modern JavaScript libraries like Redux and React perform checks checking to see if `process.env.NODE_ENV` is equal to `production` in their implementations to determine whether to run code meant for development. Since this is code that is meant to running in a web browser, the comparison returns `false` by default. However, you can use a Browserify transform like `envify` or a Webpack plugin like `DefinePlugin` to define `process.env.NODE_ENV` to equal your build process's `NODE_ENV` value or `development` by default. Setting that value to production shrunk 25% off of my total page's size (after minification and gzipping) going from 100 kilobytes to 75 kilobytes since Webpack removes unreachable code paths.

Here's what the code in a Webpack configuration could look like:

{% highlight JavaScript %}
{% raw %}
module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env': { NODE_ENV: JSON.stringify(process.env.NODE_ENV || 'development') }
  ]
}
{% endraw %}
{% endhighlight %}

Specifically, the commit I made was <https://github.com/jeffcharles/number-switcher-3000/commit/bf1e3a85337ca5caafc562bdf1ed580082e11b4b>.

The way I initially discovered this was when [Greenkeeper](http://greenkeeper.io/) tried running my continuous integration build with the new version of Redux and it failed because I'd configured my [Webdriver](http://webdriver.io/) tests to fail if it detected any non-debug log entries which is one of the ways I make sure there aren't any errors when trying an application workflow. The latest version of Redux logs a console error message if it detects that it's running minified with `NODE_ENV` not set to `production`.
