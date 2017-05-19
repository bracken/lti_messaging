# LTI Iframe `parent.postMessage` communication messages

## Messages

### lti.frameResize
```
{
  subject: "lti.frameResize",
  height: 100,
  width: 300,
  iframe_resize_id: "", // in lumen usage
  element_id: "",       // in sakai/tsugi usage
}
```

Height or width can also receive: "max"

### lti.showModuleNavigation
```
{
  subject: "lti.showModuleNavigation",
  show: false
}
{
  subject: "lti.showModuleNavigation",
  show: true
}
```
### lti.scrollToTop
```
{
  subject: "lti.scrollToTop"
}
```

### lti.navigation
```
{
  subject: "lti.navigation",
  location: "next"
}
```
location can be: previous, next, home

### lti.setUnloadMessage
```
{
  subject: "lti.setUnloadMessage",
  message: "Please don't leave me"
}
```

### lti.removeUnloadMessage
```
{
  subject: "lti.removeUnloadMessage'"
}
```

### lti.screenReaderAlert
```
{
  subject: "lti.setUnloadMessage",
  body: "Polite screen reader message"
}
```

# Implementation examples:
## Receiver (Tool Consumer)
With jquery selectors it could look something like:
```js
window.addEventListener('message', function(e) {
  try {
    var message = JSON.parse(e.data);
    switch (message.subject) {
      case 'lti.frameResize':
        var height = message.height;
        var $iframe = jQuery('#' + message.iframe_resize_id);
        if ($iframe.length == 1 && $iframe.hasClass('resizable')) {
          var height = message.height;
          if (height <= 0) height = 1;
          $iframe.css('height', height + 'px');
        }
        break;

      case 'lti.showModuleNavigation':
        if(message.show === true || message.show === false){
          $('.module-sequence-footer').toggle(message.show);
        }
        break;

      case 'lti.scrollToTop':
        $('html,body').animate({
           scrollTop: $('.tool_content_wrapper').offset().top
         }, 'fast');
        break;

      case 'lti.setUnloadMessage':
        setUnloadMessage(message.message);
        break;

      case 'lti.removeUnloadMessage':
        removeUnloadMessage();
        break;

      case 'lti.screenReaderAlert':
        $.screenReaderFlashMessageExclusive(message.body)
        break;
    }
  } catch(err) {
    (console.error || console.log).call(console, 'invalid message received from');
  }
});
```

## Sender (Tool Provider):
```js
var IframeHelper = {
  inIframe: function(){
    return self != top;
  },

  // get rid of double iframe scrollbars
  setHeight: function () {
    var default_height = $(document).height();
    default_height = default_height > 500 ? default_height : 500;

    parent.postMessage(JSON.stringify({
      subject: "lti.frameResize",
      height: default_height
    }), "*");
  },

  // get rid of lms module navigation
  hideLMSNavigation: function () {
    parent.postMessage(JSON.stringify({
      subject: "lti.showModuleNavigation",
      show: false
    }), "*");
  },

  // show lms module navigation
  showLMSNavigation: function () {
    parent.postMessage(JSON.stringify({
      subject: "lti.showModuleNavigation",
      show: true
    }), "*");
  },

  // tell the parent iframe to scroll to top
  scrollParentToTop: function () {
    parent.postMessage(JSON.stringify({
      subject: "lti.scrollToTop"
    }), "*");
  }
};

module.exports = IframeHelper;

```
# LTI Launch notes:
tsugi sends this as a launch paremeter:
```
ext_lti_element_id = tsugi_element_id_47195733i
```
and sends it up in the post message as `element_id`



lumen sends this as a query parameter to some non-lti tools:
```
iframe_resize_id = asdfasdf
```
and this is sent up in the post message as `iframe_resize_id`


# Implementations

OpenAssessments:
https://github.com/lumenlearning/OpenAssessments/blob/master/client/js/utils/communication_handler.js

https://github.com/lumenlearning/candela-utility/blob/893451ccddc9e5d80abc925a834ed08f4b84358f/themes/candela/functions.php#L102

PBJ:
https://github.com/lumenlearning/candela-utility/blob/cc8e12837fcdbef7778ddc530af9a27612e3e12d/themes/bombadil/js/iframe_resizer.js
https://github.com/lumenlearning/candela-utility/blob/893451ccddc9e5d80abc925a834ed08f4b84358f/themes/candela/functions.php#L102

Tsugi:
https://github.com/csev/tsugi-static/blob/master/js/tsugiscripts.js#L48

Canvas:
https://github.com/instructure/canvas-lms/blob/0e9ae3f5f65b7885777ffa27ef5a08747e7ac578/public/javascripts/tool_inline.js#L125



