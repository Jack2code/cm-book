# Polymer 0.5 to 1.0 Migration

标签（空格分隔）： Polymer

---
[TOC]

## Web components polyfill library
Before:
```html
<script src="bower_components/webcomponentsjs/webcomponents.min.js"></script>
```
After:
```html
<script src="bower_components/webcomponentsjs/webcomponents-lite.min.js"></script>
```
## Element registration
Before:
```html
<polymer-element name="register-me">
  <template>
    <div>Hello from my local DOM</div>
  </template>
  <script>
    Polymer();
  </script>
</polymer-element>
```
After:
```html
<dom-module id="register-me">
  <template>
    <div>Hello from my local DOM</div>
  </template>

  <script>
    Polymer({is: "register-me"});
  </script>
</dom-module>
```
## Local DOM template

Before:
```html
<polymer-element name="template-me" noscript>
  <template>
    <!-- local DOM styles --> 
    <style>
      div { color: red } 
    </style>
    <div>This is local DOM</div>
  </template>
</polymer-element>
```
After:
```html
<!-- ID attribute must match element name passed to Polymer() --> 
<dom-module id="template-me">

  <template>
    <!-- local DOM styles --> 
    <style>
      div { color: red } 
    </style>

    <div>This is local DOM</div>
  </template>

  <script>
    Polymer({is: "template-me"});
  </script>

</dom-module>
```
`<script>`标签可以放在`<dom-module>`标签里面或外面，但`</template>`标签必须在调用`Polymer`前。
## Declarative event handlers
Before:
```html
<input on-input="{{checkValue}}">
```
After:
```html
<input on-input="checkValue">
```
## Declared properties
Before:
```html
<polymer-element name="publish-me" attributes="myproperty">
```
or:
```js
Polymer({
  publish: {
    myproperty: 0
  }
});
```
After:
```js
Polymer({
  is: "publish-me",
  properties: {
    prop: Number,
    observedProp: {
      type: Number,
      value: 42,
      observer: 'observedPropChanged'
    },
    computedProp: {
      type: String,
      computed: 'computeValue(prop)'
    }
  }
});
```
## Property name to attribute name mapping
Before:
```html
<polymer-element name="map-me" attributes="fooBar">
  <script>
    Polymer({
      fooBar: ""
    });
  </script>
</polymer-element>

<map-me foobar="test1"></map-me>  <!-- sets map-me.fooBar -->
<map-me FOOBAR="test2"></map-me>  <!-- sets map-me.fooBar -->
<map-me foo-bar="test3"></map-me>  <!-- no matching property to set -->
```
After:
```
<script>
  Polymer({
    is: "map-me"
    properties: {
      fooBar: {
        type: String,
        value: ""
      }
    }
  });
</script>

<map-me foo-bar="test2"></map-me> <!-- sets map-me.fooBar -->
<map-me FOO-BAR="test3"></map-me> <!-- sets map-me.fooBar -->
<map-me foobar="test1"></map-me> <!-- no matching property, doesn't set anything on map-me -->
```
## Attribute deserialization
Before (reversed quotes accepted):
```
<my-element foo="{ 'title': 'Persuasion', 'author': 'Austen' }"></my-element>
```
After (correct JSON quotes required):
```
<my-element foo='{ "title": "Persuasion", "author": "Austen" }'></my-element>
```
## Computed properties
Before:
```
computed: {
   product: 'multiply(x,y)',
   sum: 'x + y'
},
multiply: function(a, b) {
  return a * b;
}
```
After:
```
properties: {
  product: {
    computed: 'multiply(x,y)'
  },
  sum: {
    computed: 'add(x,y)'
  }
},
multiply: function(a, b) {
  return a * b;
},
add: function(a, b) {
  return a + b;
}
```
## Property observers & Changed watchers
Before:
```
<polymer-element name="observe-prop" attributes="foo">
  <script>
    Polymer({
      foo: '',
      fooChanged: function(oldValue, newValue) {
        ...
      }
    });
  </script>
</polymer-element>
```
After:
```
Polymer({
  is: "observe-prop",
  properties: {
    foo: {
      type: String,
      value: '',
      observer: 'fooChanged'
    }
  },
  fooChanged: function(newValue, oldValue) { 
    ... 
  }
});
```

Before:
```
<polymer-element name="observe-me">
  <template>
     <my-input id="input">
  </template>
  <script>
     Polymer({
      observers: {
        'this.$.input.value': 'valueChanged'
      },
      valueChanged: function() { … } 
     });
   </script>
</polymer-element>
```
After:
```
<dom-module id="observe-me">

  <template>
     <my-input value="{{inputval}}">
  </template>

  <script>
     Polymer({
      is: "observe-me",
      properties: {
        inputval: {
          observer: 'valueChanged'
        }
      },
      valueChanged: function() { … } 
     });
   </script> 

</dom-module>
```
## Default attributes
Before:
```
<polymer-element name="register-me" checked tabindex="0" role="checkbox" noscript>
</polymer-element>
```
After:
```
hostAttributes: {
  checked: true,
  tabindex: 0,
  role: "checkbox"
}
```
## Listeners
Before:
```
<polymer-element name="magic-button" on-scroll="" on-tap="">
</polymer-element>
```
After:
```
listeners: {
  scroll: 'onScrollHandler',
  tap: 'wasTapped'
}
```
## Layout attributes replaced by layout classes and custom properties
Before:
```
<link rel="import" href="/bower_components/polymer/polymer.html">


<!-- layout attributes for the host defined on <polymer-element> -->
<polymer-element name="x-profile" layout vertical>
  <template>
    <!-- layout attributes for a local DOM element -->
    <div layout horizontal center>
      <img src="{{avatarUrl}}">
      <span class="name">{{name}}</span>
    </div>
    <p>{{details}}</p>
  </template>
  <script>
    Polymer({ ... });
  </script>
</polymer-element>
```
After:
```
<link rel="import" href="/bower_components/polymer/polymer.html">
<link rel="import" href="/bower_components/iron-flex-layout/iron-flex-layout.html">


<dom-module id="x-profile">

  <template>

    <style>   
      :host {
        /* layout properties for the host element */
        @apply(--layout-vertical);
      }
      
      .header {
        /* layout properties for a local DOM element */
        @apply(--layout-horizontal);
        @apply(--layout-center);
      }
    </style>

    <div class="header">
      <img src="{{avatarUrl}}">
      <span class="name">{{name}}</span>
    </div>
    <p>{{details}}</p>
  </template>

  <script>
    Polymer({
      is: "x-profile"
    });
  </script>

</dom-module>
```
### Using the layout classes directly
```
<head>
  ...
  <link rel="import" href="/bower_components/iron-flex-layout/classes/iron-flex-layout.html">
</head>
<body class="fullbleed layout horizontal center-center">
 ...
</body>
```
