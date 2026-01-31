title: LingoLearner - Learning and Creating Cloze Tests
date: 2026-01-31 10:03
tags: angular, wpf, apps
category: apps
slug: lingo_learner
author: Philipp Wagner
summary: Introduces LingoLearner, a small application for creating and learning with Cloze tests.

In the past week I have written lingo-learner, which is a small application for 
learning language by using cloze tests. It allows you to create you own cloze 
tests and you can run it in the web and on your desktop.

LingoLearner can be found here:

* [https://www.bytefish.de/static/apps/lingo-learner/](https://www.bytefish.de/static/apps/lingo-learner/)

All code can be found in a Git repository at:

* [https://github.com/bytefish/misc/tree/main/lingo-learner](https://github.com/bytefish/misc/tree/main/lingo-learner)

The implementation for the WPF Desktop application can be found here:

* [https://github.com/bytefish/misc/tree/main/lingo-learner-wpf](https://github.com/bytefish/misc/tree/main/lingo-learner-wpf)

Both implementations run entirely Client-side, so no data is transmitted.

## Table of contents ##

[TOC]

## Learning with LingoLearner in the Web ##

The Web application is available at:

* [https://www.bytefish.de/static/apps/lingo-learner/](https://www.bytefish.de/static/apps/lingo-learner/)


It has a pretty simple interface and starts in the "Learning mode":

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/01_lingo_learner_learn_mode.jpg">
        <img src="/static/images/blog/lingo_learner/01_lingo_learner_learn_mode.jpg" alt="Learning Mode">
    </a>
</div>

If you are uncomfortable with using English for the application, you can switch it by selecting the language 
in the upper right corner of the application:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/02_lingo_learner_select_ui_language.jpg">
        <img src="/static/images/blog/lingo_learner/02_lingo_learner_select_ui_language.jpg" alt="Select UI Language">
    </a>
</div>

You can either start with the default lesson of select the language you want to train with:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/02_lingo_learner_select_language.jpg">
        <img src="/static/images/blog/lingo_learner/02_lingo_learner_select_language.jpg" alt="Select UI Language">
    </a>
</div>

Once you have finished you lesson and clicked on Check, you can go ahead and find the next lesson to learn with:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/03_lingo_learner_select_lesson.jpg">
        <img src="/static/images/blog/lingo_learner/03_lingo_learner_select_lesson.jpg" alt="Select UI Language">
    </a>
</div>

## Creating your own lessons ##

You want to create your own lessons? It's easy by using the Admin Mode, of the app. You can find it by clicking 
on "ADMIN MODE" in the upper left corner of the application:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/02_lingo_learner_select_ui_language.jpg">
        <img src="/static/images/blog/lingo_learner/02_lingo_learner_select_ui_language.jpg" alt="Select UI Language">
    </a>
</div>

The Admin Mode has a pretty straightforward interface:


<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/04_lingo_learner_admin_mode.jpg">
        <img src="/static/images/blog/lingo_learner/04_lingo_learner_admin_mode.jpg" alt="Select UI Language">
    </a>
</div>

In the lesson title, you insert the lesson and below you are going to insert an optional description. It's useful 
to set a description for the language you are going to train with. English is always the fallback, if you didn't 
give a title in a  user interface language.

There are three elements to select from. You can add them to your lesson in the below the content elements:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/04_lingo_learner_elements.jpg">
        <img src="/static/images/blog/lingo_learner/04_lingo_learner_elements.jpg" alt="Select UI Language">
    </a>
</div>

Once the elements has been added you might want to change the position in the text. You can use up and down 
buttons on the left of the Content Element:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/05_lingo_learner_up_and_down.jpg">
        <img src="/static/images/blog/lingo_learner/05_lingo_learner_up_and_down.jpg" alt="Select UI Language">
    </a>
</div>

Sometimes you forgot to add a cloze or you want to change a sentence. You don't want to click hundreds of up 
and down buttons until you've found your element.  That's why you can expand a content element and you can 
choose an element to add after your element:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/06_lingo_learner_add_behind_element.jpg">
        <img src="/static/images/blog/lingo_learner/06_lingo_learner_add_behind_element.jpg" alt="Select UI Language">
    </a>
</div>

And if you want to train word endings, you can glue the cloze to a word, if you want to train word endings:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/10_lingo_learner_glue_cloze.jpg">
        <img src="/static/images/blog/lingo_learner/10_lingo_learner_glue_cloze.jpg" alt="Select UI Language">
    </a>
</div>

## Submitting your lesson to be included ##

If you don't want to self host the app on your server, then there is no way to directly add the lesson you have 
just created. You need to submit it to me for inclusion. I will add it to the repository and it gets loaded by 
the interface, once it has been added to the target lesson folder.

You can either submit it using a GitHub issue or you can mail it to me, using the mail address:

* `lingo-lesson@bytefish.de`

If you are using the buttons, everything is filled out. You just need to add the file:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/07_submit_proposal.jpg">
        <img src="/static/images/blog/lingo_learner/07_submit_proposal.jpg" alt="Select UI Language">
    </a>
</div>

If you want to start all over, select the "Clear All" Button next to the title.

## LingoLearner as a Desktop Application ##

If you want to use LingoLearner using a Desktop application, you can get it at:

* [[https://github.com/bytefish/misc/tree/main/lingo-learner-wpf]([https://github.com/bytefish/misc/tree/main/lingo-learner-wpf)

The interface basically is the same, except you can load and store lessons from your own filesystem. Again everything 
runs local and nothing is transmitted to an external service. 

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/07_submit_proposal.jpg">
        <img src="/static/images/blog/lingo_learner/07_submit_proposal.jpg" alt="Select UI Language">
    </a>
</div>

The interface basically is the same, except you can load and store lessons from your own filesystem. Again everything 
runs local and nothing is transmitted to an external service:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/08_lingo_learner_desktop_app.jpg">
        <img src="/static/images/blog/lingo_learner/08_lingo_learner_desktop_app.jpg" alt="Select UI Language">
    </a>
</div>

And on the left side you can see, that the buttons has changed and there is a button "Save to Local Library" now. It's 
going to store the lesson to the `C:/Temp` folder:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/09_lingo_learner_save_to_local_library.jpg">
        <img src="/static/images/blog/lingo_learner/09_lingo_learner_save_to_local_library.jpg" alt="Select UI Language">
    </a>
</div>


## Conclusion ##

And that's it.

If you want to participate in the project, it would be great to receive submissions. 👍
