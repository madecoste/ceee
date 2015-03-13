# Background #

Before reading this document, it helps to be familiar with the following:
  * The architecture behind Chrome Extensions including e.g. the process model;
  * Chrome’s architecture and its IPC and automation mechanism;
  * Chrome Frame;
  * Firefox extensibility; and
  * Internet Explorer extensibility.

# Overview #

CEEE attempts to reuse as much of the extension functionality of Chrome as possible, and adapt only the minimum parts of the environment that extensions execute in to Chrome Frame’s host browser.

In practice, the way this works out is:
  * All extension views (background page, pop-ups, infobars, etc.) continue to work exactly the same as they do in Chrome; they are rendered in an extension renderer process, they have access to the same additional JavaScript bindings that extensions running in plain Chrome do, etc.
  * Content scripts, since they must access the DOM of the host browser, are run using the host browser’s scripting engine, and access the native DOM of whatever page they are run in.
  * Extension API functions normally are dispatched within the Chrome browser process. When CEEE is using Chrome to host an extension, functions that you would expect to act on things in the host browser (e.g. tabs, windows, info bars, etc.), are instead routed over the automation interface to Chrome Frame, and handled by the CEEE code in a way appropriate to the host browser. API functions that do not act on the “browser” continue to be completed by Chrome itself, e.g. chrome.extension.getViews() which retrieves all extension views.
  * Communication between content scripts and extension views normally happens through a message router object located in the Chrome browser process. Messages are routed from whatever RenderView is on one end (either an extension view or a regular web page that has a content script in it) to whatever RenderView is on the other. CEEE extends this model so that the host browser equivalent of a regular web page with a content script can participate as an end point in this scheme.

One complicating factor is that in Internet Explorer (but not in Firefox), Chrome Frame can render the full content area of a tab if a web page opts into this functionality. To allow content scripts to also function correctly in this scenario, the CEEE architecture simply lets content script injection and content script message routing occur for such tabs just as it normally does in Chrome. This is facilitated by the architecture of Chrome Frame, since the view that Chrome Frame uses is more or less equivalent to a tab in the Chrome sense.

Extensions that use no content scripts and stick to the subset of APIs supported by CEEE should require no changes to work in IE or Firefox. However, because content scripts must run in the host browser’s scripting engine and access the host browser’s native DOM, and there are subtle differences in these between different browsers and browser versions, content scripts must be tested across all browsers an extension is meant to run on.

# Details #

We will describe in turn the Firefox and Internet Explorer designs for CEEE. We will start with the Firefox CEEE design, as it is conceptually similar to the IE CEEE but much simpler to understand.

## Firefox CEEE ##

The Firefox CEEE is implemented as a pure-JavaScript Firefox add-on, located at src/ceee/firefox.

The add-on creates a XUL toolstrip UI surface in each Firefox window. Each of these hosts a visible Chrome Frame instance in a special “privileged” mode (let’s call such a Chrome Frame a PrivilegedCF) that provides access to a few automation messages not available to web-page hosted Chrome Frame instances, and that can be used to show an extension view. There is a 1:1 relationship between Firefox windows and PrivilegedCF instances.

At any given time, a single one of these PrivilegedCF instances is designated as the “master” using a simple selection and hand-over algorithm. Essentially, the first created PrivilegedCF is master until its destruction is triggered by the Firefox window it is attached to closing, at which point an arbitrary one of the other windows’ PrivilegedCF instances is designated as master.

Whenever a PrivilegedCF is designated as master, an automation message is sent on that PrivilegedCF’s automation channel. This message lists the extension API functions that should thus be routed to this PrivilegedCF instance.

When the master PrivilegedCF receives extension API function calls (all of which are asynchronous), it implements them in terms of native Firefox functionality, and sends back an asynchronous response when appropriate. Function calls and responses are formatted as JSON strings.

Upon loading, each window’s PrivilegedCF sends an automation message to retrieve information on which extension is installed, from Chrome. It then loads the manifest file to retrieve information about that extension’s content scripts and CSS files that should be injected into pages based on URL patterns. It then applies rules that are identical to Chrome’s content script URL matching rules to determine which pages to inject these scripts and CSS files into.

When a page loads in the window a PrivilegedCF is attached to (or was already loaded at the time the PrivilegedCF was created), the PrivilegedCF will evaluate the script, prefixed with some bootstrapping JavaScript, into the page, within a JavaScript sandbox. The script sees a safe, wrapped view of the DOM of the page. The bootstrapping JavaScript contains the same JavaScript bindings used in Chrome for content scripts, plus some Firefox-specific adapter JavaScript. This gives it access to content script messaging and the handful of other content script-accessible extension APIs.

## Internet Explorer CEEE ##

The architecture of the IE CEEE is conceptually very similar, but in practice much more complex than the Firefox CEEE.  See the following overview diagram:

![http://ceee.googlecode.com/files/ceee_ie.png](http://ceee.googlecode.com/files/ceee_ie.png)

In the diagram above, we can draw the following parallels to the Firefox CEEE by way of explaining the architecture:
  * All Chrome Frame instances are the ActiveX equivalent of the PrivilegedCF instances we mentioned in Firefox; however:
  * Instead of a 1:1 relationship between browser windows and PrivilegedCF instances, there is 1 PrivilegedCF per Internet Explorer tab (hosted by an Internet Explorer Toolband add-on) to show UI, plus 1 invisible PrivilegedCF used for content script messaging (hosted by an Internet explorer Browser Helper Object or BHO add-on); and
  * The CEEE Broker is a COM server, that registers an elevation policy that allows low integrity processes to invoke on it. The PrivilegedCF that it hosts is the equivalent of the “Master” PrivilegedCF in the Firefox CEEE. All extension API function calls are routed to this PrivilegedCF, which delegates their fulfillment to one or more BHOs and/or one or more Frame Executors. The CEEE Broker is also the central point for translating IE event notifications to Chrome Extension event notifications.
  * Each BHO lives on the thread of a browser tab. It is used to fulfill actions that pertain to the given tab, to inject content scripts into pages hosted in the tab, and to subscribe to event notifications from IE that pertain to the tab or content within it, and forward those events to the CEEE Broker.
  * A Frame Executor object is injected into each thread that owns an IEFrame top-level browser window. It is used to fulfill actions that pertain to the given window.
  * Content scripts are injected by the BHOs into an ActiveScript scripting engine, identical to but separate from the scripting engine used to run scripts that are part of the web page. As in the Firefox case, they receive the same JavaScript bindings as are used for content scripts in Chrome, with some additional adapters to make those bindings work correctly in IE.

### CEEE Broker Details ###

The broker process plays host to all overtaken extension APIs, as well as being a center point for event distribution.

It contains a registry of executors associated to threads in which they run. Some executors are provided by the BHO instances that runs in the tab threads. Others are instantiated by the executor creator which injects them in the thread where we don't have a BHO running (e.g., the threads for the frame windows).

The Broker contains a non-visible PrivilegedCF instance, which is solely used to do API and event-related messaging.

The executor registry and the API execution are hosted in the COM multi-threaded apartment, to allow concurrent execution of registrations and API execution, and to avoid re-entrancy issues inherent with single-threaded COM apartments. But the API execution and event handling are all posted to a single worker thread so that we don't need to protect against concurrent access when handling asynchronous API invocations.

The PrivilegedCF instance and other inherently single-threaded execution, such as e.g. shell access and shell notification processing (e.g. for bookmark change notifications once we implement that API) is performed in a single-threaded apartment in this process. So API invocation response and events to be propagated to Chrome Frame are posted to this single-threaded apartment using a Postman COM object instantiated in there.

The CEEE Broker process is accessible by browser helper objects running in both Medium and Low integrity. Since it straddles security contexts in this fashion, it is important to restrict the set of interfaces it exposes, and to exercise caution in all implementation.

To this end, all interfaces it exposes will verify the information provided against the invoking security context, to avoid providing an elevation path from Low Integrity for e.g. DoS or spoofing attacks.

The lifetime of this process is managed by having the BHO and Toolband instances maintain a reference on it during their lifetime.