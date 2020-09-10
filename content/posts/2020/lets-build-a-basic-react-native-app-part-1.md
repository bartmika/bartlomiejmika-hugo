---
title: "Let's Build a Basic React Native App - Part 1"
date: 2020-08-28T10:38:31-04:00
draft: true
categories:
- "development"
tags:
- "react-native"
- "javascript"
---

Let's build a simple React Native App from scratch! This post is part of a x part series detailing how to build a app called **"Garbage Day"**. The first post will discuss *installation* and *navigation*.

<!--more-->
Before we start building, let's make sure we are on the same page with our understanding.

* I assume you've just started learning **React Native**.
* I assume you have used **React (web)** before and are proficient with it.
* I assume you are using a **MacOS**. If you are using **Linux** or **Windows** then I assume you will make the extra effort to figure out how to work with it there to follow along.
* I assume you have a basic understanding of the **Linux Terminal** and you know a few commands.

This post is broken down into 4 sections: Analysis, Design, and Implementation.

# 1. Analysis
## 1.1. Goal
The purpose of this app is to allow users to lookup the city zoning area they belong to, for garbage collection, and have the app notify the user when "Garbage Day" arrives. In essence this is app a digital **Garbage Collection Schedule**.

## 1.2. Data
Where will we get out **Garbage Collection Schedule** data? Check your municipalities website and see if you can find any. I've found ours via the following link: [City of London Garbage Open Data - Collection Schedule](https://opendata.london.ca/datasets/02417efe16254c09ab7bfcb72e2496d5).

If you are unable to find your schedule, contact your municipality and if no luck then follow along with what we are using.

## 1.3. Learning Resources
Please spend some time looking up some resources to help you. I have found the following sites useful for me:

* [Official React Native Developer Documenation](https://reactnative.dev/)
* [Offical Expo Develpoer Documenation](https://docs.expo.io/)
* [Learning React Native: where to start](https://hackernoon.com/learning-react-native-where-to-start-49df64cf14a2)
* [React Native Express - Learn React Native, the cross-platform app framework](http://www.reactnativeexpress.com/)
* [React Native Tutorial](https://www.tutorialspoint.com/react_native/index.htm)

## 1.3. Installation & Setup

If you have not read the [installation instructions](https://reactnative.dev/docs/environment-setup) from the main React Native site then please check it out now. In essence there are three development environments you can use:

1. [Snack](https://snack.expo.io/) - If you want to run all your code in a browser, you can!
2. [Expo](https://docs.expo.io/) - If you want to build a mobile app powered by HTML / JS then this is the solution to use.
3. [React Native](https://reactnative.dev/docs/environment-setup) - If you want to have a native app compiled to a phone, this is the solution you want.

Let's see what's involved in all three steps.

### 1.3.1. Development Method 1: Snack
Unfortunately not to much to say here - simply log in and start coding!

### 1.3.2. Development Method 2: Expo
Install the command line tools

```bash
$ npm install --global expo-cli
```

Afterwords go to your development directory.

```bash
$ cd ~/javascript/github.com/bartmika
```

And create our app:

```bash
$ expo init garbageday
```

If you get the error ``-bash: expo: command not found`` then, according to [this discussion](https://stackoverflow.com/questions/51775496/exp-command-not-found-how-do-i-add-expo-cli-to-path), run the following command which will let you know where ``expo`` is located:

```bash
$ npm bin -g
```

You should get an output like ``/usr/local/lib/node_modules/node/bin/expo``. Therefore knowing this, run the following:

```bash
$ /usr/local/lib/node_modules/node/bin/expo init garbageday
```

Finally, go into your app folder start the project by running:

```bash
$ cd garbageday
$ expo start garbageday
```

Congratulations you have setup the project.

### 1.3.3. Development Method 3: React Native
Begin by going to your development directory.

```bash
$ cd ~/javascript/github.com/bartmika
```

And create our app:

```bash
$ npx react-native init garbageday
```

Go into our app folder and run it (Note: I am using a Mac):

```bash
$ cd garbageday
$ npx react-native run-ios
```

If you see this then congratulations, you have setup the project successfully!

![Wireframe](/img/2020/08/28/welcome_to_react.png)

# 2. Design
I want to make this app dead simple. Here is a quick mock up created using a software package called [Pencil](https://pencil.evolus.vn/).

![Wireframe](/img/2020/08/28/garbageday_wireframes.png)

I imagine it will be something like this:

* When app loads up for the first time, it will load the "Home Screen (Initial)".
* The home screen will inform the user they need to set their zone. User can click "Settings" button to be redirected to the "Settings Screen".
* When "Setting Screen" loads, user can only select "London" from the city picker and a pre-populated list of letters in the "Zone" picker.
* Once you click "Save" , the app will navigate to the "Main Page" where you can see the date of the next garbage pickup.
* If use clicks the "Settings" button, it will take them to the "Settings Page" again.

# 3. Development
For development let's take method 3 and use the **React Native** project build.

### 3.1 Navigating Between Screens
Let's follow the instructions on the [official developer documentation](https://reactnative.dev/docs/navigation).

Install the dependencies:

```bash
$ npm install --save @react-navigation/native @react-navigation/stack
$ npm install --save react-native-reanimated react-native-gesture-handler react-native-screens react-native-safe-area-context @react-native-community/masked-view
```

For iOS with bare React Native project, make sure you have Cocoapods installed. Then install the pods to complete the installation:

```bash
$ cd ios
$ pod install
$ cd ..
```

You have now installed all the dependencies to run our project.

The next step is the following:

* We need to add the ``react-native-gesture-handler`` to top of the ``App.js`` file.
* Wrap the whole app in ``NavigationContainer``.

Therefore your ``App.js`` should look like this:

```javascript
//
// src/App.js
//
import 'react-native-gesture-handler';
import * as React from 'react';
import { NavigationContainer } from '@react-navigation/native';

const App = () => {
  return (
    <NavigationContainer>
      {/* Rest of your app code */}
    </NavigationContainer>
  );
};

export default App;
```

If we continue with the developer documentation then we will have the following contents. I have gone through and added my comments - follow along by reading all the comments from (1) to (x):

```javascript
//
// src/App.js
//
import 'react-native-gesture-handler'; // (1) Required as per docs. Must be at top!

// (2) Standard libraries for our project.
import * as React from 'react';
import {
  Text,
  Button,
} from 'react-native';

// (3) This is the new library we are using for navigation. Where can we learn more about this library? We can
//     learn more about it via the developer docs at https://reactnavigation.org/docs/getting-started. According
//     to the developer docs:
//
// (a) "createStackNavigator" is a function that returns an object containing 2 properties: Screen and Navigator.
//     Both of them are React components used for configuring the navigator. The Navigator should contain Screen
//     elements as its children to define the configuration for routes.
// (b) "NavigationContainer" is a component which manages our navigation tree and contains the navigation state.
//     This component must wrap all navigators structure. Usually, we'd render this component at the root of our
//     app, which is usually the component exported from `App.js`.
//
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

// (4) Initialize the navigator which is stack-based.
const Stack = createStackNavigator();

// (5) We declare a function-based component with the `navigation` variable returned from the props using ES6 `destructuring` feature. As you can see we utilize the `navigation` variable to use the `navigate` function to redirect the user to our
// profile screen. Note: "Profile" is the ID of the screen.
const HomeScreen = ({ navigation }) => {
  return (
    <Button
      title="Go to Jane's profile"
      onPress={() =>
        navigation.navigate('Profile', { name: 'Jane' })
      }
    />
  );
};

// (6) Here is another screen.
const ProfileScreen = () => {
  return <Text>This is Jane's profile</Text>;
};

// (7) The is the main entry into our application. We need to structure out `App` component differently if we want
//     to support navigation. We need to encompass the app inside a "NavigationContainer" component and then reference
//     all the screens we have using the `Stack`.
//
//     "HomeScreen" is a component which has all our homepage logic and GUI. We reference it through "Home" ID.
//     "ProfileScreen" is a component which has all our profile logic and GUI. We reference it through "Profile" ID.
//     "title" allows us to set what to display in the navigation title.
//     
const App = () => {
  return (
    <NavigationContainer>
       <Stack.Navigator>
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: 'Welcome' }}
        />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default App;
```

### 3.2 Container-Component Pattern

Now that we have a boilerplate to work from, let's modify the code to conform to the [container-component pattern](https://reactpatterns.com/#container-component). In addition, modify the app according to the design of our app.

Our new folder structure will look as follows (please note I am only including relevant files):

```
garbageday
│   App.js
│
└───containers
│   │   HomeContainer.jsx
│   │   SettingComponent.jsx
│
└───components
│   │   HomeComponent.jsx
│   │   SettingComponent.jsx
```

We would get the following code:

###### App.js

```javascript
//
// App.js
//
import 'react-native-gesture-handler';
import * as React from 'react';

import {
  Text,
  Button,
} from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

import HomeContainer from "./containers/HomeContainer";
import SettingContainer from "./containers/SettingContainer";

const Stack = createStackNavigator();

const App = () => {
  return (
    <NavigationContainer>
       <Stack.Navigator>
        <Stack.Screen
          name="Home"
          component={HomeContainer}
          options={{
              title: 'Garbage Day'
          }}
        />
        <Stack.Screen
          name="Settings"
          component={SettingContainer}
          options={{
              title: 'Settings',
              headerBackTitle: "Back",
          }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default App;
```

###### containers/HomeContainer.js

```javascript
//
// containers/HomeContainer.js
//
import React from 'react';

import HomeComponent from '../components/HomeComponent';


class HomeContainer extends React.Component {

    /**
     *  Initializer
     *------------------------------------------------------------
     */

    /**
     *  Utility
     *------------------------------------------------------------
     */

    /**
     *  Component Life-cycle Management
     *------------------------------------------------------------
     */

    /**
     *  API callback functions
     *------------------------------------------------------------
     */

    /**
     *  Event handling functions
     *------------------------------------------------------------
     */

    /**
     *  Main render function
     *------------------------------------------------------------
     */

    componentDidMount() {}

    render () {
        return (
            <HomeComponent
                navigation={this.props.navigation}
            />
        )
    }
}

export default HomeContainer;
```

###### components/HomeComponent.js

```javascript
//
// components/HomeComponent.js
//
import * as React from 'react';

import {
  Text,
  Button,
} from 'react-native';

const HomeComponent = (props) => {
    const { navigation } = props;
    return (
    <Button
      title="Go to Settings"
      onPress={() =>
        navigation.navigate('Settings', { name: 'Jane' })
      }
    />
    );
};


export default HomeComponent;
```


###### components/SettingContainer.js

```javascript
//
// containers/SettingContainer.js
//
import React from 'react';

import SettingComponent from '../components/SettingComponent';


class SettingContainer extends React.Component {

    /**
     *  Initializer
     *------------------------------------------------------------
     */

    /**
     *  Utility
     *------------------------------------------------------------
     */

    /**
     *  Component Life-cycle Management
     *------------------------------------------------------------
     */

    /**
     *  API callback functions
     *------------------------------------------------------------
     */

    /**
     *  Event handling functions
     *------------------------------------------------------------
     */

    /**
     *  Main render function
     *------------------------------------------------------------
     */

    componentDidMount() {}

    render () {
        return (
            <SettingComponent
                navigation={this.props.navigation}
            />
        )
    }
}

export default SettingContainer;
```


###### components/SettingComponent.js

```javascript
//
// components/SettingComponent.js
//
import * as React from 'react';

import {
  Text,
  Button,
} from 'react-native';

const SettingComponent = (props) => {
    const { navigation } = props;
    return (
        <>
            <Text>Welcome to settings</Text>
        </>
    );
};


export default SettingComponent;
```

Now if we run our app, you should see something as simple as the following:

![App](/img/2020/08/28/app1.png)

Great job!
