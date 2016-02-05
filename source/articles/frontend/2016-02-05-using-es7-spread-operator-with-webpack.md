---
title: Using the ES7 Spread Operator with Webpack
category: Frontend
---

## The Problem

I'm currently evaluating Redux as a means to tidy up a React app, and in
following the examples in the docs came across the spread operator (`...`).

In the following code the spread operator copies in all of the keys and values
of the existing state, and allows us to extend it with a new value for `bold`:

```
const comment = (state, action) => {
  switch (action.type) {
    case 'COMMENT_TOGGLE_BOLD ':
      return {
        ...state,
        bold: !state.bold
      }
    default:
      return state
  }
}
```

Problem is, while I have set up Webpack 6 to succesfully transpile and bundle
JSX and ES6 (ES2015), it fails to recognise this use of the spread operator as
at time of writing it is still in the proposal stage:

```
Module build failed: SyntaxError: /.../assets/js/reducers/comments.js:
Unexpected token (9:8)
   7 |
   8 |       return {
>  9 |         ...state,
     |         ^
  10 |         bold: !state.bold
  11 |       }
  12 |     default:
```

## A Quick Solution

The quickest solution I found to resolve this, after trying to add in specific
plugins like `transform-object-rest-spread`, was to enable the `stage-2` preset:

```
npm install --save babel-preset-stage-2
```

And then in `webpack.config.js`:

```
module: {
  loaders: [
    {
      test: /\.jsx?$/,
      exclude: /node_modules/,
      loader: 'babel-loader',
      query: {
        presets:[ 'es2015', 'react', 'stage-2' ]
      }
    }
  ]
}
```

Be aware that the additional operators and behaviour which this preset brings in
may be subject to change--in this case I think the benefits of the spread
operator in reducing boilerplate override those concerns, as the operator
semantics appear likely to become standard before long.
