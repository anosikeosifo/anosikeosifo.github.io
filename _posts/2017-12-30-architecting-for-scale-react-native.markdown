---
layout: post
title:  "React Native - Architecting for scale"
date:   2017-12-30
categories: react-native
excerpt_separator: <!--more-->
comments: true
---

---

This is an approach to architecting large React Native applications - a great one i suppose.

Learning and building with React Native(RN) has been an interesting one to say the least, with the fast pace of innovation and changes in the ecosystem (don't forget, we're not at version 1.0 yet!).

I started out building breakable toys and soon grew into the need to build a large (enterprise scale) application - and then I got stuck. Where to place what? How to bring them all together, and all those issues you'd usually face when architecting a solution from the ground up.

Before we get into the details, I'll highlight the tools that influenced this choice of architecture:

- [react-navigation](https://www.npmjs.com/package/react-navigation) - Application navigation
- [redux](https://www.npmjs.com/package/redux) - Aplication state management
- [redux-thunk](https://www.npmjs.com/package/redux-thunk) - Enabling asynchronous dispatching of actions
- [jest](http://facebook.github.io/jest/) - Javascript testing
- [reselect](https://www.npmjs.com/package/reselect) - Selector library for Redux
- [axios](https://www.npmjs.com/package/axios) - Http Client
- [fastlane](https://fastlane.tools/) - Automation tool

### How I'd learnt to structure application modules.

![type-based architecture]({{site.baseurl}}/assets/type_based_arch.png){:class="img-responsive"  width="500"}
<!--more-->

The above structure (type-based) had been my go-to for previous light-weight apps, as it was easy to reason about and app components seemed very visible, so i started out with it.
However, as the scope of the application began to grow I found myself including more files to the predefined modules. In no time, each module became crowded with numerous unrelated files; navigation between modules, while keeping mental note of implementation flow became a pain, and clearly not feasible.



## A better approach:

![type-based architecture]({{site.baseurl}}/assets/feature_based_arch.png){:class="img-responsive" width="500"}

Because the other files & folders (some truncated) are usually part of a default `react-native init` installation, our focus would be on the *`src`* folder:

### **fastlane/**

This folder, as you must have noticed, exists on same level as *`src`*. It contains configuration for deployment as well as other app release related tasks. See the [fastlane website](https://fastlane.tools/) for more details.


### **src/**

As seen in the screenshot, let's discuss the rationale behind this architecture


### **api/**
![type-based architecture]({{site.baseurl}}/assets/api_dir.png){:class="img-responsive" width="500"}

This folder contains logic related to external API communications, it includes:

- `constants.js` - where all related static values are stored.
- `helper.js` - for storing reusable logic.
- individual feature files -  Each feature file would contain api communication logic for a particular feature.


#### **assets/**
Just as the name implies, this houses static files (e.g images) used in the application.


#### **components/**

![type-based architecture]({{site.baseurl}}/assets/components_dir.png){:class="img-responsive" width="500"}

Shared components that are used across features are placed in this directory. An example of such (as shown above) is the `layout` component, which is used to wrap the application components and influence the app's overall layout.




#### **features/**
![type-based architecture]({{site.baseurl}}/assets/features_dir.png){:class="img-responsive" width="500"}

A major part of this architecture, the `feature` folder, consists of individual modules for each of the application's feature.

Let's examining the **`/explore/`** module in more details, it consists of:

##### */actions*
Like in most react/react-native application. this folder would contain the Action Creators for this feature.

##### */components*
Here we place component and style logic for the various aspects of the explore feature.

##### */containers*
Redux-related logic is placed in the containers folder. For this use-case there's only a single source of truth for direct interaction between the redux logic and the `explore feature`.

Here's what the `/containers/index.js` looks like:

{% highlight text %}

import { connect } from "react-redux";
import { bindActionCreators } from "redux";
import Explore from "../components/explore"; //imports the feature's entry component.
import { navigateToLogin } from "navigation/actions"; //imports navigation action to be used by feature
import { getExploreData } from "../actions"; // imports action creators as needed.
import { getCategoryListing } from "../selectors"; // imports selectors as needed.

const mapStateToProps = state => ({
  exploreData: state.exploreData,
  categoryListing: getCategoryListing(state)
});

const mapDispatchToProps = dispatch =>
  bindActionCreators(
    {
      getExploreData
    },
    dispatch
  );

// Connects the entry-component and makes it the default export.
export default connect(mapStateToProps, mapDispatchToProps)(Explore);

{% endhighlight %}

##### */reducers*
Each feature has a reducer that modifies its own slice of the application state. All reducers are later merged using redux's `combineReducers` function.

##### */selectors*
This might come across as a bit strange to some of us, however this aspect of our architecture is influenced by the [reselect package](https://www.npmjs.com/package/reselect), which enables us to efficiently compute dreived data from our application's state.

In this architecture, we grouped the application selectors on a feature-basis so as to make interaction between various aspects of the app easy to reason about. More details on how reselect works can be found [here](https://www.npmjs.com/package/reselect).


##### */constants.js*
This file contains static values used within the feature. An example of what we could store here is  `ACTION_TYPES` data.


#### **navigation/**
![navigation]({{site.baseurl}}/assets/navigation_dir.png){:class="img-responsive" width="500"}

This is another component of our architecture slightly influenced by a [react-package](https://www.npmjs.com/package/react-navigation)

Due to the scale of this application, we wouldn't be relying on the out-of-the-box navigation with `this.props.navigation`, rather we would be be connecting the app's navigation to the overall application state. This means that our redux-backed store would be aware of the navigation state.

Lets explore the roles of the various directories in this module.

##### */actions*
This contains logic for a bunch of navigation-specific action-creators, thus the **`actions/index.js`** content would mostly look like:

{% highlight text %}

import { NavigationActions } from "react-navigation";
import * as screenNames from "../screen_names";

export const navigateToLogin = () =>
  NavigationActions.navigate({
    routeName: screenNames.LOGIN
  });

export const navigateToSplash = () =>
  NavigationActions.navigate({
    routeName: screenNames.SPLASH
  });
{% endhighlight %}

As you might have observed, we're importing the **`NavigationActions`** object from the [react-navigation](https://www.npmjs.com/package/react-navigation) package to implement native navigation; they provide [a really great documentation](https://reactnavigation.org/docs/) too!

##### */containers*
This is the where we connect our navigation logic to the application state - using **`mapStateToProps`** and **`mapDispatchToProps`**.
The logic of this container component would look somewhat like this:

{% highlight text %}

//imports the navigators we've defined for our app
import ApplicationNavigator from "../navigators";

//used in connecting the app's navigation data to redux
import { addNavigationHelpers } from "react-navigation";

//takes the navigation slice of state and maps it to a prop, so we can used it around the application.
const mapStateToProps = state => ({
  navigation: state.navigationData
});

const mapDispatchToProps = dispatch => ({ dispatch });

class ApplicationNavigatorContainer extends Component {
  .
  .
  .

  render() {
    return (
      <ApplicationNavigator
        navigation={addNavigationHelpers({
          dispatch: this.props.dispatch,
          state: this.props.navigation
        })}
      />
    );
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(
  ApplicationNavigatorContainer
);

{% endhighlight %}


##### */navigators*
The entry point of this module is the **`index.js`** file, and here's what it looks like:

{% highlight text %}
...
import { StackNavigator } from "react-navigation";
import AuthNavigator from "./authentication";
import MainNavigator from "./main";
import * as screenNames from "../screen_names";
import Splash from "features/splash/containers";

const appNavigator = StackNavigator({
  [screenNames.SPLASH]: {
    screen: Splash
  },

  [screenNames.LOGIN]: {
    screen: AuthNavigator
  },

  [screenNames.MAIN]: {
    screen: MainNavigator
  }
});

export default appNavigator;

{% endhighlight %}

This entry point navigator files serves as the root navigator for the whole application, and is what is imported in the **`/containers/index.js`** file in the previous section.

It aggregates all of the navigators for various scenes in the application and links them up to their entry routes. Besides aggregation, it also routes to individual scenes when appropriate (as seen with the ***Splash*** route in the snippet above).

At the top of this code, we see that other scene-based navigation are imported. A scene navigator could like this (an excerpt from the imported ***MainNavigator***).

{% highlight text %}
export default TabNavigator(
  {
    [screenNames.EXPLORE]: {
      screen: Explore
    },

    [screenNames.USER_PROFILE]: {
      screen: Explore
    }
  },
  {
    tabBarOptions: {
      swipeEnabled: false,
      tabBarPosiion: "bottom",
      activeTintColor: "#ff5a5f",
      style: {
        backgroundColor: "#fff",
        paddingVertical: 7
      }
    }
  }
);

{% endhighlight %}

You can have several scene-based navigators as required by your application. what's key is to import them into the parent navigator and connect to the appropriate route.


##### */reducers*
Because our app's navigation data now takes a slice of the application state, we would need a reducer to properly update this sliced based on triggered actions. Compared to other reducers in our app, this has a specialised implementation for both initial state and the reducer function:

{% highlight text %}

let initialState = {
  index: 0,
  routes: [
    {
      key: "initialLogin",
      routeName: <initial_screen_name>
    }
  ]
};

const navigationReducer = (state, action) => {
  const nextState = ApplicationNavigator.router.getStateForAction(
    action,
    state
  );

  return nextState || state;
};

export default navigationReducer;

{% endhighlight %}


#### **reducers/**
This is the application-level reducer. Its function is to merge the various feature-level reducers using redux's **`combineReucers`** function.

Here's what **`reducers/index.js`** looks like:

{% highlight text %}

import { combineReducers } from "redux";
import eventData from "features/events/reducers";
import exploreData from "features/explore/reducers";
import navigationData from "navigation/reducers";

export default combineReducers({
  eventData,
  exploreData,
  navigationData
});

{% endhighlight %}


#### **styles/**
This module holds our application-level styles. which are then referenced in individual components using React Native's multiple-style syntax shown in the example below:

{% highlight text %}
<View styles={[ globalStyles.wrapper, styles.textWrap ]}>
  ...
</View>
{% endhighlight %}


#### **index.js**

This is the application entry file, it wraps the application with the Layout we earlier discussed, and connects the app to the store using the redux's **`Provider`** higher order component.

It looks like this:

{% highlight text %}

export default class LargeApp extends Component {
  render() {
    return (
      <Provider store={myStore}>
        <Layout>
          <ApplicationNavigator />
        </Layout>
      </Provider>
    );
  }
}

{% endhighlight %}


#### **myStore.js**
this contains the store-creation logic.



Whoa, This must have been quite a long read!

While this isn't all-encompassing*, I hope you find it insightful and helpful. I would be glad if you could share your thoughts and opinions on this as well as on how you go about implementing large-scale RN apps.

\*In order to focus on the core of the article's purpose, i avoided going into details on the various **`__tests_`** modules.




### Additional Resources:
* [React Navigation](https://reactnavigation.org)
* [How to better organize your React applications?](https://medium.com/@alexmngn/how-to-better-organize-your-react-applications-2fd3ea1920f1)
* [Structuring a React Native + MobX Application the Right Way](https://blog.geekyants.com/structuring-a-react-native-mobx-application-the-right-way-c1e9d2ae0ff7)

Some Great React Native Resources
* [React Native - Getting Started](https://facebook.github.io/react-native/docs/getting-started.html)
* [React Native with Fastlane and Crashlytics Beta](https://medium.com/@tjkangs/react-native-with-fastlane-and-crashlytics-beta-aa0d6ca630fd)
