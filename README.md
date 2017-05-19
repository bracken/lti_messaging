# LTI Iframe `parent.postMessage` communication messages

This is documenting a `window.postMessage` communication mechanism 
that is available in the Tool Consumers Canvas and Sakai, and is 
used by many tool providers to improve the user experience of LTI
tools.

How it works: The Tool Provider (TP) is embedded in an Iframe inside
the Tool Consumer (TC). Using Javascript's `window.postMessage` the
TP can send a message to the TC via the browser. This allows the
systems to communicate about practical UI things like the size the
Iframe should be, and various navigation conveniences.

In current usage messages are only sent from TP to TC, but the same
mechanism can be used to send messages from TC to TP.

## Messages

### lti.frameResize
This message is sent to let the TC know the desired Iframe size of
the TP. This is useful to get rid of double scrollbar situations.

There is no guarantee that the TC will honor the desired size. The
TC is likely to have a max size it allows, or make other decisions
it deems best.

The TP may want to debounce or throttle how many of these events it
sends if the content tends to change height frequently.
```js
{
  subject: "lti.frameResize",
  height: 100,
  width: 300,           // not currently in use
  iframe_resize_id: "", // in lumen usage
  element_id: "",       // in sakai/tsugi usage
}
```
Height or width can also receive `max` which means the TC should
try to set the Iframe to use all the available space in the window.

### lti.showModuleNavigation
Some TPs have internal navigation that can be confused with the
navigation available in the TC. This message lets the TC know it
should hide its module navigation if it can.

This might not make sense in some TCs and "module navigation" may
be ambiguous for some. The intent is to hide navigation UI items
that might cause confusion. So it's up to the TC to decide what 
that means.

An example usage is a quiz tool that has buttons to go to the next
and previous questions. Those buttons may easily be confused with
the buttons to go to the next/previous module item in an LMS. So
the TP may ask the LMS to hide its navigation buttons during the
quiz, but send a message to show them again once the quiz is over.

```js
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
If the TP has expanded the height of its Iframe and the user is
scrolled down in the content and links to a new page, the 
default browser behavior is to keep the display scrolled down
where it currently is instead of jumping to the top like the
user expects when they click a link.

This message allows the TP to let the TC know to scroll up in
those situations.

The TP may want to debounce or throttle how many of these 
events it sends.

```js
{
  subject: "lti.scrollToTop"
}
```

### lti.navigation
If the TC is OK hiding its navigation items, it may want to
allow the TP to still control the TC's navigation a bit.

The TC can decide what locations it want to allow the TP to
choose. The simple examples are for going to next/previous
module items in an LMS.

```js
{
  subject: "lti.navigation",
  location: "next"
}
```
location can be: `previous`, `next`, `home`

### lti.setUnloadMessage
An Iframe can't prevent a user from navigating away in the
outer window. In some cases, like while taking a quiz or
editing some content, the TP can let the TC know to not
let the user navigate away without seeing the provided
message.

```js
{
  subject: "lti.setUnloadMessage",
  message: "Please don't leave me"
}
```

### lti.removeUnloadMessage
Use in conjunction with `lti.sentUnloadMessage`. When the TP
is done with it activity it should let the TC know to remove
the unload message.

```js
{
  subject: "lti.removeUnloadMessage'"
}
```

### lti.screenReaderAlert
Screen readers have the ability to receive an alert from
Javascript but they sometimes can't handle such an event
happening within an Iframe. This message can be used to
allow a TP to send a message to the screen reader via the
TC.

```js
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

Tsugi: 

https://github.com/csev/tsugi-static/blob/master/js/tsugiscripts.js#L48

Canvas: 

https://github.com/instructure/canvas-lms/blob/0e9ae3f5f65b7885777ffa27ef5a08747e7ac578/public/javascripts/tool_inline.js#L125

Sakai:

Documentation:
https://github.com/sakaiproject/sakai/blob/8b1817a36a1da551527b41c09297ed3a7d7ca2b3/basiclti/docs/POSTMESSAGE.md

https://github.com/sakaiproject/sakai/blob/8cf21791daaa0e36b95fbc0a531937f4c2caee16/library/src/webapp/js/headscripts.js#L244

https://github.com/sakaiproject/sakai/search?utf8=%E2%9C%93&q=lti.frameResize&type=

OpenAssessments:

https://github.com/lumenlearning/OpenAssessments/blob/master/client/js/utils/communication_handler.js

https://github.com/lumenlearning/candela-utility/blob/893451ccddc9e5d80abc925a834ed08f4b84358f/themes/candela/functions.php#L102

PBJ:

https://github.com/lumenlearning/candela-utility/blob/cc8e12837fcdbef7778ddc530af9a27612e3e12d/themes/bombadil/js/iframe_resizer.js

https://github.com/lumenlearning/candela-utility/blob/893451ccddc9e5d80abc925a834ed08f4b84358f/themes/candela/functions.php#L102

