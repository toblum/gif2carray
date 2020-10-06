# gif2carray
Animated GIF to Arduino converter for [PxMatrix](https://github.com/2dom/PxMatrix)


## Background
This page converts animated GIF files to Arduino byte-arrays in RGB565 format as used by the great PxMatrix library in the [examples](https://github.com/2dom/PxMatrix/tree/master/examples/black_lives).

I couldn't make the converter scripts work in my environment, so I implemented this

## Usage
Make sure the entered urls can be remote loaded.
The conversion is done completely in the browser, no entered data is transferred to the server. See the [source code](https://github.com/toblum/gif2carray/tree/main) for implementation details.

It's just a quick'n'dirty implementation, but it may help you too. Implementation is done using vite/vue3 and the [gifuct-js](https://github.com/matt-way/gifuct-js) library