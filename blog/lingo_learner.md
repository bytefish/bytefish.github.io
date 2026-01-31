title: LingoLearner - Learning and Creating Cloze Tests
date: 2026-01-31 10:03
tags: angular, wpf, apps
category: apps
slug: lingo_learner
author: Philipp Wagner
summary: Introduces LingoLearner, a small application for creating and learning with Cloze tests.

In the past week I have written lingo-learner, which is a small application for 
learning a language by using cloze tests. It allows you to create your own cloze 
tests.

You can run it in the web and on your desktop.

LingoLearner can be found here:

* [https://www.bytefish.de/static/apps/lingo-learner/](https://www.bytefish.de/static/apps/lingo-learner/)

All code can be found in a Git repository at:

* [https://github.com/bytefish/misc/tree/main/lingo-learner](https://github.com/bytefish/misc/tree/main/lingo-learner)

The implementation for the WPF Desktop application can be found here:

* [https://github.com/bytefish/misc/tree/main/lingo-learner-wpf](https://github.com/bytefish/misc/tree/main/lingo-learner-wpf)

Both implementations run entirely client-side on your machine. No data is transmitted to external services.

## Table of contents ##

[TOC]

## Learning with LingoLearner in the Web ##

The Web application is available at:

* [https://www.bytefish.de/static/apps/lingo-learner/](https://www.bytefish.de/static/apps/lingo-learner/)

It has a pretty simple interface and starts in the "Learning mode":

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/notes/lingo-learner.jpg">
        <img src="/static/images/notes/lingo-learner.jpg" alt="Lingo Learner">
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

You can select the lesson to train in the top bar:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/03_lingo_learner_select_lesson.jpg">
        <img src="/static/images/blog/lingo_learner/03_lingo_learner_select_lesson.jpg" alt="Select UI Language">
    </a>
</div>

If you click on "Check" button, your Cloze test is evaluated:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/10_lingo_learner_check_results.jpg">
        <img src="/static/images/blog/lingo_learner/10_lingo_learner_check_results.jpg" alt="Select UI Language">
    </a>
</div>

## Creating your own lessons ##

You want to create your own lessons? It's easy by using the Admin Mode!

You can find it by clicking on "ADMIN MODE" in the upper left corner of the application:

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

In the upper area, you can insert the lesson title and descriptions for each language. The description is 
optional, but it's useful to give instructions for the lesson.

English is always the fallback, if you didn't fill out a description.

There are three elements to select from. You can find them in the section below the content elements:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/04_lingo_learner_elements.jpg">
        <img src="/static/images/blog/lingo_learner/04_lingo_learner_elements.jpg" alt="Select UI Language">
    </a>
</div>

Once the element has been added, you might want to change the position in the text. You can use up and down 
buttons on the left side of the Content Element:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/05_lingo_learner_up_and_down.jpg">
        <img src="/static/images/blog/lingo_learner/05_lingo_learner_up_and_down.jpg" alt="Select UI Language">
    </a>
</div>

But you don't want to click the up button a hundred times to move it around.

That's why you can expand a content element and add it behind a given element:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/06_lingo_learner_add_behind_element.jpg">
        <img src="/static/images/blog/lingo_learner/06_lingo_learner_add_behind_element.jpg" alt="Select UI Language">
    </a>
</div>

If you want to train word endings, you can glue the cloze to a word:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/10_lingo_learner_glue_cloze.jpg">
        <img src="/static/images/blog/lingo_learner/10_lingo_learner_glue_cloze.jpg" alt="Select UI Language">
    </a>
</div>

## Submitting your lesson to be included ##

If you don't want to self host the app on your server, there is no way to directly add the lesson to the website.

You'll need to submit it to me for inclusion. 

I will add it to the repository and it gets loaded by the interface, once it has been added to the target lesson folder.

Either submit it using a GitHub Issue or send it by mail to:

* `lingo-lesson@bytefish.de`

If you are using the buttons, the lesson is downloaded and the GitHub Issue or e-Mail is filled out already. 

You'll just need to attach the downloaded file:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/07_submit_proposal.jpg">
        <img src="/static/images/blog/lingo_learner/07_submit_proposal.jpg" alt="Select UI Language">
    </a>
</div>

Now if you want to start all over, select the "Clear All" Button next to the title.

## LingoLearner as a Desktop Application ##

If you want to use LingoLearner using a Desktop application, you can get it at:

* [https://github.com/bytefish/misc/tree/main/lingo-learner-wpf]([https://github.com/bytefish/misc/tree/main/lingo-learner-wpf)

The interface basically is the same, except you can now load and store lessons on your own filesystem. 

Again everything runs local and nothing is transmitted to an external service.

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/08_lingo_learner_desktop_app.jpg">
        <img src="/static/images/blog/lingo_learner/08_lingo_learner_desktop_app.jpg" alt="Select UI Language">
    </a>
</div>

And on the right side of the application you can see, that the buttons has changed and there is a button 
"Save to Local Library" now. 

It's going to store the lesson to the `C:/Temp` folder and instantly reload all lessons:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/lingo_learner/09_lingo_learner_save_to_local_library.jpg">
        <img src="/static/images/blog/lingo_learner/09_lingo_learner_save_to_local_library.jpg" alt="Select UI Language">
    </a>
</div>

## Conclusion ##

And that's it.

If you want to participate in the project, it would be great to receive submissions. 👍
