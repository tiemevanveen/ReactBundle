Using ReactBundle
===================

*Where we explain how to install and start using ReactBundle*

Installation
------------

First and foremost, note that you have a complete example with React, Webpack and Symfony Standard Edition at [Limenius/symfony-react-sandbox](https://github.com/Limenius/symfony-react-sandbox) ready for you. Feel free to clone it, run it, experiment, and copy the pieces you need to your project. Being this bundle a frontend-oriented bundle, you are expected to have a compatible frontend setup.

### Step 1: Download the Bundle

Open a command console, enter your project directory and execute the
following command to download the latest stable version of this bundle:

    $ composer require limenius/react-bundle

This command requires you to have Composer installed globally, as explained
in the *installation chapter* of the Composer documentation.

### Step 2: Enable the Bundle

Then, enable the bundle by adding the following line in the `app/AppKernel.php`
file of your project:

```php
// app/AppKernel.php

// ...
class AppKernel extends Kernel
{
    public function registerBundles()
    {
        $bundles = array(
            // ...

            new Limenius\ReactBundle\LimeniusReactBundle(),
        );

        // ...
    }

    // ...
}
```

### Step 3: (optional) Configure the bundle

The bundle comes with a sensible default configuration, which is listed below. If you skip this step, these defaults will be used.

    limenius_react:
        # Other options are "only-serverside" and "only-clientside"
        default_rendering: "both"
        serverside_rendering:
            # In case of error in server-side rendering, throw exception
            fail_loud: false
            # Replay every console.log message produced during server-side rendering
            # in the JavaScript console
            # Note that if enabled it will throw a (harmless) React warning
            trace: false
            # Location of the server bundle, that contains React and React on Rails.
            # null will default to `app/Resources/webpack/server-bundle.js`
            server_bundle_path: null

## JavaScript and Webpack Set Up

In order to use React components you need to register them in your JavaScript. This bundle makes use of the React On Rails npm package to render React Components (don't worry, you don't need to write any Ruby code! ;) ).

Your code exposing a react component would look like this:

```js
import ReactOnRails from 'react-on-rails';
import RecipesApp from './RecipesAppServer';

ReactOnRails.register({ RecipesApp });
```

Where RecipesApp is the component we want to register in this example.

Note that it is very likely that you will need separated entry points for your server-side and client-side components, for dealing with things like routing. This is a common issue with any universal (isomorphic) application. Again, see the sandbox for an example of how to deal with this.

If you use server-side rendering, you are also expected to have a Webpack bundle for it, containing React, React on Rails and your JavaScript code that will be used to evaluate your component.

Take a look at [the webpack configuration in the symfony-react-sandbox](https://github.com/Limenius/symfony-react-sandbox/blob/master/webpack.config.serverside.js) for more information.

If not configured otherwise this bundle will try to find your server side JavaScript bundle in `app/Resources/webpack/server-bundle.js`

## Start using the bundle

You can insert React components in your Twig templates with:

```twig
{{ react_component('RecipesApp', {'props': props}) }}
```

Where `RecipesApp` is, in this case, the name of our component, and `props` are the props for your component. Props can either be a JSON encoded string or an array. 

For instance, a controller action that will produce a valid props could be:

```php
/**
 * @Route("/recipes", name="recipes")
 */
public function homeAction(Request $request)
{
    $serializer = $this->get('serializer');
    return $this->render('recipe/home.html.twig', [
        'props' => $serializer->serialize(
            ['recipes' => $this->get('recipe.manager')->findAll()->recipes], 'json')
    ]);
}
```

## Server-side, client-side or both?

You can choose whether your React components will be rendered only client-side, only server-side or both, either in the configuration as stated above or per twig tag basis.

If you set the option `rendering` of the twig call, you can override your config (default is to render both server-side and client-side).

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'client-side'}) }}
```

Will render the component only client-side, whereas the following code

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'server-side'}) }}
```

... will render the component only server-side (and as a result the dynamic components won't work).

Or both (default):

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'both'}) }}
```

You can explore these options by looking at the generated HTML code.

if you don't need to set any options you can also pass in the props directly: 

```twig
{{ react_component('RecipesApp', props) }}
```

## Debugging

One imporant point when running server-side JavaScript code from PHP is the management of debug messages thrown by `console.log`. ReactBundle, inspired React on Rails, has means to replay `console.log` messages into the JavaScript console of your browser.

To enable tracing, you can set a config parameter, as stated above, or you can set it in your template in this way:

```twig
{{ react_component('RecipesApp', {'props': props, 'trace': true}) }}
```

Note that in this case you will probably see a React warning like

*"Warning: render(): Target node has markup rendered by React, but there are unrelated nodes as well. This is most commonly caused by white-space inserted around server-rendered markup."*

This warning is harmlesss and will go away when you disable trace in production. It means that when rendering the component client-side and comparing with the server-side equivalent, React has found extra characters. Those characters are your debug messages, so don't worry about it.

## Performance with Server-Side rendering

Server-side rendering should be used in applications where you can cache the resulting HTML using Varnish or something similar. Otherwise, as for every request the server bundle containing React must be copied either to a file (if your runtime is node.js) or via memcpy (if you have the V8Js PHP extension enabled) and re-interpreted, this can have an overhead. Note that would be theoretically possible to precompile the server bundle in the V8Js object, but as after every request is destroyed, due to the stateless nature of most PHP applications, this is not possible in practice. Thus the components that cannot be cached are best rendered client-side.


## Redux

If you're using [Redux](http://redux.js.org/) you could use the bundle to hydrate your store's:

Use `redux_store` in your twig file before you render your components depending on your store:

```twig
{{ redux_store('MySharedReduxStore', initialState ) }}
{{ react_component('RecipesApp') }}
```
`MySharedReduxStore` here is the identifier you're using in your javascript to get the store. The `initialState` can either be a JSON encoded string or an array. 

Then, expose your store in your bundle, just like your exposed your components:

```js
import ReactOnRails from 'react-on-rails';
import RecipesApp from './RecipesAppServer';
import configureStore from './store/configureStore';

ReactOnRails.registerStore({ configureStore });
ReactOnRails.register({ RecipesApp });
```

Finally use `ReactOnRails.getStore` where you would have used your the object you passed into `registerStore`.

```js
// Get hydrated store
const store = ReactOnRails.getStore('MySharedReduxStore');

return (
  <Provider store={store}>
    <Scorecard />
  </Provider>
);
```

Make sure you use the same identifier here (`MySharedReduxStore`) as you used in your twig file to set up the store. 
