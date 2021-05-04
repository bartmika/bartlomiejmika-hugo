---
title: "How to Structure a React Project"
date: 2020-08-28T10:38:31-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "React"
- "Javascript"
---

![](/img/2020/08/28/joan-gamell-ZS67i1HLllo-unsplash.jpg)

In the article you will get the boilerplate structure I use for [#React](/tags/react/) projects.

<!--more-->

Here is the simple example of code you can copy and paste into your project and then refactor.

The design pattern we are primarily going to use is the [container-component](https://reactpatterns.com/#container-component).

Here is the **container:**

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

    constructor(props) {
        super(props);
    }

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

Here is the **component:**

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

Some notes from the above code:

* Containers are *Class based Components*.
* Components are *Function based Components*.
* The container and component file exist in a single directory.
* I always keep the comments, even if I don't use the section.

Cover photo by [Joan Gamell](https://unsplash.com/@gamell) on [Unsplash](https://unsplash.com).
