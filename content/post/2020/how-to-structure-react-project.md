---
title: "How to Structure a React Project"
date: 2020-08-28T10:38:31-04:00
draft: true
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "React"
- "Javascript"
---

TODO



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
