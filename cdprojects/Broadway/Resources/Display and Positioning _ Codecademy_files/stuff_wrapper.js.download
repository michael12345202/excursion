;(function (win, undefined) {
  'use strict';

  window.__webMiniPreview = true;

  // Reference for faster access.
  var doc  = win.document
    , body = doc.querySelector('body')

    // Work around an IE9 bug.
    , JSON = win.JSON

    // Child iframe that will run the code.
    , iframe

    // Secret to use for communication with parent window.
    , secret;

  // Remove the old iframe and create a new one.
  function reset () {
    if (iframe) body.removeChild(iframe);
    iframe = doc.createElement('iframe');
    iframe.setAttribute('width', '100%');
    iframe.setAttribute('height', '100%');
    body.appendChild(iframe);
  }

  //  Disabling window functions
  //  For disabling interruptive functions like alert and prompt

  var disabledFunctions = [];

  function setDisabledFunctions (names) {
    disabledFunctions = names;
  }

  function disableFunctions (doc) {
    var names = disabledFunctions;
    doc.write('<' + 'script>')
    for (var i = 0; i < names.length; i ++) {
      doc.write('window.' + names[i] + ' = function(){};'); // Overrwrite it with noop
    }
    doc.write('</' + 'script>')
  }

  function handleAnchorTags(doc) {
    var anchors = doc.querySelectorAll('a');
    for (var i = 0; i < anchors.length; i++) {
      var anchor = anchors[i];
      var href = anchor.getAttribute('href');
      anchor.onclick = function(e) {
        e.preventDefault();
        post('clickLink', href);
      }
    }
  }

  // Create an iframe and load html into it. Post back with a `load` event
  // when the iframe has loaded.
  function load (html) {
    reset();
    iframe.addEventListener('load', function () {
      post('load', null);
    }, false);
    var doc = iframe.contentWindow.document;
    doc.open();
    disableFunctions(doc)

    var docWriteChecks = 0;
    var docWriteCheckInterval = setInterval(function () {
      if (doc.querySelector('body')) {
        handleAnchorTags(doc);
        clearInterval(docWriteCheckInterval);
      } else if (docWriteChecks > 20) {
        clearInterval(docWriteCheckInterval);
      }
      docWriteChecks++;
    }, 500);

    doc.write(html);
    doc.close();
  }

  // Helper to post back to the parent window.
  function post (type, data) {
    var msg = JSON.stringify({
      data   : data
    , type   : type
    , secret : secret
    });
    win.parent.postMessage(msg, '*');
  }

  // Helper to the `evaljs` function to report back `err` and `res`.
  function report (err, res) {
    var ContextError = iframe.contentWindow.Error;

    // If it's an error object we want to get it's information to be able to
    // reconstruct it in the parent window.
    // Safari and Opera doesn't differentiate between context scoped errors
    // while others do. 
    if ( 
         err                         && 
         err instanceof Error        ||
         err instanceof ContextError
       ) {

      // Pull up information from the `Error` object to send to the parent window.
      err = {
        message       : err.message
      , stack         : err.stack
      , type          : err.type
      , 'arguments'   : err['arguments']

      // `__errorType__` is the [[Class]] of the error. Could be a native or subclass of it.
      , __errorType__ : String(err.constructor).trim().match(/^function ([^\(\s]+)/)[1]
      };
    }

    // If the result is a funciton then get a string representation.
    // TODO: Look into a simple stringification library.
    if (typeof res === 'function') res = String(res);
    post('evaljs', {
      error  : err
    , result : res
    });
  }

  // Eval javascript in our iframe context.
  function evaljs (js) {
    var res = null;

    try {
      res = iframe.contentWindow['eval'](js);
    } catch (e) {
      report(e, null);
      throw e;
    }
    report(null, res);
  }

  // Post back our current html.
  function html () {
    post('html', iframe.contentWindow.document.documentElement.outerHTML);
  }

  // All the actions available to the outside world.
  var actions = {
    load   : load
  , evaljs : evaljs
  , html   : html
  , disableFunctions: setDisabledFunctions
  };

  // Handle messages from parent window.
  win.addEventListener('message', function (e) {
    var msg;
    try {
      msg  = JSON.parse(e.data);
    } catch (err) {

      // We are only concerned in JSON messages.
      return;
    }
    var type = msg.type
      , data = msg.data;

    // We expect all messages to have a secret to make sure
    // it's a stuff.js message.
    if (!msg.secret) return;
    if (!secret && msg.type === 'handshake') {

      // Set the current secret.
      secret = msg.secret;
    } else if (msg.secret !== secret) {
      return;
    } else {

      // Route message to the correct action.
      actions[type](data);  
    }
  }, false);

  // Export an emit funciton on this window that could be accessible to the
  // iframe context to emit events to the parent window.
  win.stuffEmit = function (event, data) {
    post(event, data);
  };

})(window);
