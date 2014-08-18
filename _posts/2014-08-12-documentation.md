---
layout: post
title:  Documentation
date: 2014-08-11 23:21:31
disqus: y
---

## PROJECT IDEA

######- Pre-GSoC State of Art:

AChecker validates the static html of a web page given as input against the WCAG 2.0 guidelines. Basically it takes the source code of the input URL and validates it according to the specified WCAG accessibility norms.<br/><br/>
However, some accessibility issues can not be identified just by this static validation. For e.g a lot of HTML elements within a webpage are triggered by Document Object Model (DOM) events which are unaccounted for.

######- Purpose & Scope of the Project:

The main crux is to improve the checks done by AChecker, by taking the current WCAG 2.0 guidelines implemented as a black box, and extend the current AChecker implementation to validate the dynamically generated Web content too. <br/><br/>

---

## DESIGN & ARCHITECTURE

**Aim**: Fetching newly generated dynamically triggered HTML from the webpage upon a certain set of events (triggers) and then validate.  
<br/>
Basic steps carried out:  
<br/>  

- Javascript and jQuery scripts to detect the common events that manipulate DOM and cause generation of new HTML contents.
- Using PhantomJS and CasperJS libraries (discussed in detail below), to push all such events into an array and evaluate each of them.
    - Thereafter, fetch the new HTML content upon triggering all such dynamic events.
- Merge the new HTML contents obtained from each trigger with the original source code (without duplication).
- Pass this newly obtained HTML for AChecker validation and get union of the results obtained. Thus, the final result of AChecker will be validation of all such dynamic events including the default static HTML source (Integration with AChecker)
<br/>

**Crux**: The above task of fetching the dynamically generated new HTML contents requires the site to be opened and an event to be triggered.
The javascript written in the source of the webpage for a particular event triggered needs to get rendered.   

###### - Architectural Requirements

To accomplish the above said, we need a headless browser Webkit i.e. a browser framework without the actual UI which renders the JavaScripts and is able to modify DOM => [PhantomJS](phantomjs.org)  
PhantomJS in itself has DOM handling and jQuery selector functionality. However, to ease the usage of PhantomJS and a better hand at navigation scripting, we use [CasperJS](http://casperjs.org/) too. Its basically an utility written for PhantomJS which provides high-level functions for mamipulating DOM and remotely accessing it, the most apt library for this project.  

<br/>

**Setting up the environment**
<br/>
*For installing PhantomJS:*  

Using the native package manager (apt-get for Ubuntu and Debian, pacman for Arch Linux, pkg_add for OpenBSD, etc).  

e.g.: for Ubuntu  
`$ sudo apt-get install phantomjs`  
Click [here](http://phantomjs.org/download.html) for more information  
<br/>

*For installing CasperJS:*  
Using npm:  

`$ npm install -g casperjs`  

Click [here](http://docs.casperjs.org/en/latest/installation.html) for detailed instructions and alternative methods  
<br/>

---

## IMPLEMENTATION

**Developer Manual**  
###### - Detecting events that trigger and manipulate DOM contents

Basically since CasperJS is an utility framework in Javascript which allows accessing remote DOM, events that manipulate DOM are detected using standard jQuery selectors and the HTML entities are returned back to CasperJS environment.  
<br/>
To get a gist, following aptly describes the process being carried out  
<br/>
![CasperJS framework](http://docs.casperjs.org/en/latest/_images/evaluate-diagram.png "CasperJS framework")

The __evalute()__ function acts as a gate between the CasperJS environment and the webpage opened. Thus, everytime a closure is passed to __evaluate()__, we enter the page and execute code as if using the browser console.  

<br/>
For e.g., let there be a function __getOnClickTriggerElements()__:  
<br/>

```javascript
function getOnClickTriggerElements(){
    /* .. Javascript/jQuery HTML entity selector code here .. */
    return onClickTriggerElements;
    }
```  
<br/>
This function returns a certain set of HTML elements which are capable of manipulating DOM and generating new HTML content:  


###### - Evaluating each trigger element separately

Now the above function is evaluated as:  
<br/>

```javascript
onClickTriggerElements = casper.evaluate(getOnClickTriggerElements);
```  

<br/>
Here, __onClickTriggerElements__ represents array of HTML entities in CasperJS environment returned from __evaluate()__ function. This array contains list of HTML elements which when triggered, manipulate DOM and generate new HTML contents respectively.
Thus similar functions (such as onClick, input Forms, mouseover and related mouse events, button triggers, etc) are written which fetch such DOM manipulating elements and return them in the form of an array.  

Now that all the trigger elements are returned in an array, all these elements need to be triggered one-by-one and then the newly generated HTML content can be fetched accordingly.  
<br/>
Following the CasperJS evaluation architecture described above, we trigger each of these elements (using __casper.each()__ since each element of the array is to be triggered separately) in the __evaluate()__ function and render the respective newly generated HTML content.  
Assumption: Currently, a __wait()__ function is used assuming it takes atmost 1 sec to load the DOM after triggering an element. Thereafter, the source code at that particular instant is captured and sent back to CasperJS environment from the __evaluate()__ function. For each of the trigger, a HTML file with name __`data<counter>.html`__ is generated (counter represents no. of such data files) which contains source code of the webpage at the point after triggering an element respectively.  
<br/>
A sample code snippet for this would seem like:  
<br/>

```javascript
// wait for approx. 1000 ms to load the DOM
casper.wait(1000, function(){
        HTMLSource = this.evaluate(function(){
            return document.getElementsByTagName('html')[0].outerHTML;
            });
        // save the HTMLSource contents into separate files for each of the triggers 
        });
```  

###### Merge different new HTML contents generated by different triggers (without duplication).

**Crux**: Since in the part discussed above, each of the DOM manipulating element is triggered iteratively, the HTML source code grows incrementally with duplication. Consider following illustration:  
<br/>

Say there are 4 trigger elements on a webpage.

- Now the _data0.html_ file contains the static source code of the webpage being validated.
- After processing 1st trigger, say some new HTML content gets generated and thus, _data1.html_ contains a snapshot of the source code of the webpage after triggering 1st dynamic element.
- After processing 2nd trigger, _data2.html_ contains source code after triggering 2nd element. However, it also includes the HTML content generated by 1st trigger since this is an iterative process and the HTML/DOM is kept triggering continually.  
    - A fresh reload is not done after every trigger because loading the webpage after every trigger would prove to be costly in terms of time.  

<br/>
Also assuming that the last HTML generated (say _dataN.html_) would contain all of the newly generated HTML alongwith original source code is not correct since some triggers may overwrite the content written by other.  
<br/>
**Aim**: Get all the newly generated HTML contents by each of the trigger (Maximization) considering even slightest trigger and merge them.  
<br/>
**Implementation**:  
So, basically following iterative approach, a __diff__ of adjacent HTML files is taken using

```
diff -u data0.html data1.html
```
which provides an output in the form of ```git diff```. Iteratively fetching all the '+' differences from the ```diff``` output would give the dynamic HTML content generated by all the triggers scattered across different HTML files. All such positive diffs are merged into a file say **dynamicDOMElements.html**. (File _mergeFiles.py_ does the work described)
<br/>


## Integration with AChecker

Our objective is to get the union of this newly formed merged dynamicDOMElements file and the original source code of the webpage, and thereafter pass this whole HTML content to AChecker for validation.  
However, while getting the union of dynamicDOMElements with source code, there needs to be some HTML headers associated with dynamic content else it would lead to false validation of that content via AChecker stating some problem types of HTML headers with dynamically obtained HTML content although it might be alright in the original source code.  
In a nutshell, false negatives regarding HTML headers( ```<!DOCTYPE>``` headers and ```<html>``` attributes) w.r.t to validating new content must not be given as output by AChecker.  
<br/>

Now that _dynamicDOMElements_ file is created, instead of unifying the webpage source code with dynamic content disjointly with manually inputing DOCTYPE and HTML headers, we perform a _selective merge_ contents of this file with the main source code.  
Here selective merge refers that all this dynamically generated content must be placed above ```</body></html>``` tags, thus preserving the original DOCTYPE and HTML headers of the webpage. Thus now _mergedSourceContent.html_ (say) contains a merged HTML code which contains source code of original webpage selectively merged with dynamically generated content while preserving the headers (avoiding false negatives).  
<br/>
**Result**:
Thus, with performing above steps now we have _mergedSourceContent.html_ file with merged contents which were generated by triggering DOM manipulating elements. Also, this file contains apt DOCTYPE headers and html attributes (same as that of the original webpage), thus leading to no ambiguous warnings/problems reporting from AChecker.
Now that _mergedSourceContent.html_ is generated, the task of integration breaks down into following:  
<br/>
On getting the URL from the input form, if the URL contains no errors, then a _execute.sh_ script is called which contains a sequence of steps to be done

- Calls the CasperJS script with the URL as a parameter and stores the dynamic content fetched from each trigger into _HTMLSourceFiles_ folderwith filenames as _data0.html_, _data1.html_, _data2.html_ and so on.
- Thereafter python script _mergeFiles.py_ is called and _dynamicDOMElements.html_ gets generated with all the dynamic contents merged into one file.
- After this gets done, a selective merge of this dynamic content and static source code of the webpage is done (using some bash commands). This results in generation of _mergedSourceContent.html_ file now containing the webpage source code merged with dynamic content.
- Replacing the ```$validate_content``` variable: Thereafter, the content to be validated is read from the above generated file instead of directly taking the static source code from the web and thus is loaded into ```$validate_content``` variable.
    - With help of some switches, we fetch contents of the _mergedSourceContent.html_ file if the "Show Source" option is enabled in the options menu while validating the URL (i.e. the source code of the webpage to be validated along with dynamic content is to be shown).

###### Testing

The above implemented has been tested thoroughly on a sample site built for reference and debugging purposes hosted [here](http://web.iiit.ac.in/~tejas.shah/gsoc/casper/sampleMultipleForms.html). The site mentioned is of simplistic form but contains minimal required features for triggering. It contains 4 DOM manipulating events which generate new HTML content. Codebase has been made rigorous enough to tackle such elements within other sites.
After being completely built, it has been tested against various sites which gives additional known, likely, potential problems accordingly to the HTML content they generate. Results have been discussed and found satisfactory.

###### Screenshots
SampleHTML validation comparison.. Full image [here](http://web.iiit.ac.in/~tejas.shah/gsoc/screenshots/comparison1.jpg) 

![Comparison sample HTML](http://web.iiit.ac.in/~tejas.shah/gsoc/screenshots/comparison1.jpg)

--

Google.com validation comparison.. Full image [here](http://web.iiit.ac.in/~tejas.shah/gsoc/screenshots/comparison2.jpg)  

![Comparison Google.com](http://web.iiit.ac.in/~tejas.shah/gsoc/screenshots/comparison2.jpg)

###### MISCELLANEOUS

For doing standalone work, a local github repo is maintained. [Link-to-local-repo-used](https://github.com/tejasshah93/gsoc-code)  
<br/>
Since the implementation discussed above requires reading, writing, modification of files via Apache hosted server, its necessary to give required permissions to the 'AChecker/checker' folder. i.e. basically giving Apache server the ownership of the files ('apache' user in Fedora and alike, whereas 'www-data in Ubuntu and similar')  
<br/>

Following commands should do the work:

In Fedora:

```
sudo chown -R apache:apache <folder-name>  
```
<br/>

In Ubuntu:

```
sudo chown -R www-data:www-data <folder-name>
```
<br/>

--- 

###### Contact Me

For any queries or just to get in touch:  
<br/>
Tejas Shah  
Email ID: tejas.urwelcome@gmail.com  
github  : tejasshah93  
IRC nick: jash4/carver404

---
