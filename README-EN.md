# General specifications about the development of training modules on My Training Platform

- Generalities
- Structuration of a module
- Particular case of a SCORM module
- Communication with the LMS

## Generalities

The LMS runs thanks to an UNIX system and assets (including modules) are hosted on AWS S3.

That's why files are `case-sensitive` and have to respect web's standards (no special characters, no spaces, no capital letters). Cf https://ed.fnal.gov/lincon/tech_web_naming.shtml

Files' content can be reduced to optimize display results.

Inside the LMS, the module is displayed in an iframe of 100% of width and height, with a burger menu on the top left corner.

The first 50 pixels on the top left musn't be used because they would be recovered by the burger.


## Structuration of a module

The module has to be delivered compressed in an archive ZIP without a password.

Files composing the module can be prioritised and can count as many folders and subfolders as you need.

Inclusions of files have to use relative paths.

The first html file must be called `index.html`

WARNING : you must compressed the content beginning with the bottom of the website (from index.html) and not beginning with an upper level. This means that if you store your files into a project folder you mustn't compress this project folder but the files inside of it.

All required ressources for the proper functioning of the module have to be included inside of it. There is no external call for HTTP.


## Particular case of a SCORM module

The LMS integrates a SCORM 2004 standard and modules which can benefit from it.

In the case of a scorm module, the first file doesn't have to be named index.html but it has to be specify in the manifest (lmsmanifest.xml).

It is recommended to use an API wrapper, like this one : https://github.com/pipwerks/scorm-api-wrapper/blob/master/src/JavaScript/SCORM_API_wrapper.js


Code example to connect to LMS

```javascript

var scorm = pipwerks.SCORM;  //Shortcut
var scormStartTime = (new Date()).getTime();
var lmsConnected = false;

/*
 * Utils
 */
function handleError(msg) {
  // how you want to handle error, for example "alert"
}

/*
 * Connection managment
 */
function connectToLMS() {
  // scorm.init returns a boolean
  lmsConnected = scorm.init();

  // If the course couldn't connect to the LMS for some reason...
  if (lmsConnected == false) {
    // add a body class
    var a = document.body;
    a.classList ? a.classList.add('no-lms') : a.className += ' no-lms';
    // ... let's alert the user
    handleError("Error: Course could not connect with the LMS");
  }
}

function disconnectFromLMS() {
  // If the lmsConnection is active...
  if (lmsConnected) {
    // Log session time
    var now = (new Date()).getTime();
    var milliseconds = now - scormStartTime;
    var seconds = Math.round(milliseconds/1000);
    var interval = 'PT' + seconds + 'S';
    var success = scorm.set('cmi.session_time', interval);
    // If the value wasn't successfully set...
    if (!success) {
       // ... let's alert the user
       handleError("Error: Session Time could not be set");
    }
    scorm.quit();
    lmsConnected = false;
  }
}

window.onload = connectToLMS;
window.onunload = disconnectFromLMS;

```

Once connected it's possible to get values from the LMS or to attribute them using scorm.set(key, value) or scorm.get(key).

## Communication LMS's specific features

You can interact with the LMS to mark the module as completed, to note a score and/or post in the community.

We make available the file `mtp_lms.js` which does the connection between the LMS and your module.

It is available here : https://github.com/semiodesign/MTP_LMS

Two methods are available :

> `createCommunityPost` owns an object as a variable. This objects can have for attributes :

- `community_group_id`, integer, optional
  - If the `community_group_id` isn't filled, the post is created on the user's profile and visible by his friends.
  - In the case of a `community_group_id` filled, the post is related to the relevant group.
- `content`, string, optional
  - The text content of the post which will appear in the community.
- `image`, string, optional
  - An image linked to the post. It has to be converted into string base64.
  - We recommend you to use toDataURL() (cf: https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toDataURL) to do it.
- `callback`, function, optional
  - The name of the function which will process dates returned by the LMS.

> `moduleWon` takes an object as a variable. This object can have for attributes:

- `success`, boolean, require
  - Save the module as succeeded or not for the current user.
- `callback`, function, optional
  - Name of the function which will process datas returned by the LMS.

If a callback is filled, it has to take an object (the one returned by the LMS) as a parameter.

The callback of `createCommunityPost` returns:

```javascript

{ event: 'create_community_post',
 status: 'OK',
 status_message: '',
 community_group_url: '<url_du_groupe>',
 show_me_url: '<url_profil_utilisateur>',
 root_url: 'root_lms_url'
}

```

The callback of `moduleWon` returns:

```javascript

{ event: 'create_training_module_completion',
 status: 'OK',
 status_message: '',
 root_url: 'root_lms_url'
}

```

An easy error handling is set up.
If the treatment goes well, the key `status` of the returned object is `OK` and the key `message_status` is empty.
Otherwise, the key `status` of the returned object is `ERROR` and the key `message_status` allows you to retrieve more specific datas about the nature of the mistake.

Example of use:

```javascript
var group_id = 17;
var message = 'Happy Message';
var image = 'my_base64_image';
var submit = submitButton;
var lms = new MTP_LMS();

var communityPostAdded = function(obj) {
  if (obj.status === 'ERROR') {
    // Process the error
  } else {
    // Offer to see the group in which the post has been sent
  }
};

var moduleCompletionSaved = function(obj) {
  if (obj.status === 'ERROR') {
    // Process the error
  } else {
    // Thank for playing
  }
};

submit.on('click', function() {
  lms.createCommunityPost({
    content: message,
    community_group_id: group_id,
    callback: communityPostAdded
  });

  lms.moduleWon({
    success: true,
    callback: moduleCompletionSaved
  });
});

```