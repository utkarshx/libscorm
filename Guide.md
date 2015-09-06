It is possible to pick and choose parts of the library. Section 1 below contains several standalone examples using functionality from `library/core`. If you don't want custom control, skip to section three where you will learn how to use the SCORM content template to create basic SCOs.



# 1 Core Library Classes #

This section illustrates how to use each core class individually. Let's start with the simplest example: discovering the learner's name and displaying a greeting.

## 1.1 Example: Hello, world ##

Every SCO requires the files `LMS.js` and `LibScormException.js`; these
files are located in `library/core` and suffice to connect to SCORM.

Hello, world! (Actually, hello <learner name>)

```
<html>
<head>
	<script language="JavaScript" type="text/javascript"
		src="LibScormException.js"> </script>
	<script language="JavaScript"
		type="text/javascript" src="LMS.js"> </script>
	<script language="JavaScript" type="text/javascript">
	var lms = null;

	function load() {
		try {
			lms = new LMS(window);
			document.getElementById("message").innerHTML =
				"Hello, " + lms.GetValue("cmi.learner_name");
		} catch(e) {
			alert(e.toString());
		}
	} function unload() {
		if(lms) lms.Terminate();
	}
	</script>
</head>
<body onload='load()' onunload='unload()'>
	<p id="message">Not connected to LMS</p>
</body>
</html>
```

## 1.2 LMS Class Methods ##

Constructor(window)
> Scans window's parents for SCORM API.
Terminate()
> Ensure your SCO calls this method before exit.
IsTerminated()
> Boolean.
GetValue(name),
SetValue(name, value)
> 'name' is an element of the CMI data model.
Commit()
> Alerts the LMS that the data sent by SetValues should be
> committed to persistent storage. Some LMSes use this as a cue
> to update the course table of contents or other sequencing
> interface elements.
StartSessionTimer(),
PauseSessionTimer(),
RecordSessionTime()
> Writes the value of the Session Timer stopwatch to
> cmi.session\_time.
GetLastError()
> Returns SCORM error code of last error, or 0.
> Discouraged. Rely on exception handling in JavaScript
> SCOs. This method is included only for Flash SCOs (which
> can't catch exceptions).
GetDiagnostic()
> Returns explanation for last error. Discouraged.
GetErrorString()
> Discouraged.

## 1.3 Example: Objectives and Interactions ##

They are both instances of "CMI Bags" -- CMI collections with no specified
order, and are manipulated similarly in LibSCORM. The library caches bag
values, so repeated access will not result in slow duplicate LMS calls.

Setting objective score and success status

```
<html>
<head>
	<script language="JavaScript" type="text/javascript"
		src="LibScormException.js"> </script>
	<script language="JavaScript"
		type="text/javascript" src="LMS.js"> </script>
	<script language="JavaScript"
		type="text/javascript" src="CMIBag.js"> </script>
	<script language="JavaScript" type="text/javascript">
	var lms = null;
	var obj = null;

	function load() {
		try {
			lms = new LMS(window);
			obj = new CMIBag(lms, "objectives");
		} catch(e) {
			alert(e.toString());
		}
	} function unload() {
		if(lms) {
			// Set primary objective
			lms.SetValue('cmi.score.scaled', '1');
			lms.SetValue('cmi.success_status', 'passed');
			// Set secondary
			obj.SetValue('secondary_name', 
			               'success_status', 'failed');
			obj.SetValue('secondary_name', 
			               'score.scaled', '0.25');
			lms.Terminate();
		}
	}
	</script>
</head>
<body onload='load()' onunload='unload()'>
	<p>This SCO is successful but secondary objective
	   'secondary_name' is not.
	</p>
</body>
</html>
```

## 1.4 CMIBag Class Methods ##

Constructor(lms, bagname)
> 'bagname' is a CMI element e.g. 'objectives'
> or 'interactions', or a sub-bag such as
> 'interactions.n.objectives'.
GetValue(id, elem)
> 'elem' is e.g. 'success\_status'.
SetValue(id, elem, val)
> Creates ID if necessary.
GetElementList()
> Returns array of existing IDs.
GetIndex(id)
> Indices may change without warning between learner
> sessions. Returns -1 if ID does not yet exist.

## 1.5 Example: Inter-SCO Sequencing ##

Although performing such sequencing requests is a matter of a few
SCORM API calls, doing so is tricky because it will terminate the
SCO. Some parts of your code may not know others' pre-termination cleanup
requirements, hence shouldn't terminate the SCO indiscriminately. Using
the `InterScoSeq` class, you can establish an event handler to be called
before termination.

Be careful not to call `Terminate()` more than once. The following
example checks `IsTerminated()` to avoid duplicating termination invoked
by inter-sco sequencing. The example will do nothing interesting outside
of a content package.

```
<html>
<head>
	<script language="JavaScript" type="text/javascript"
		src="LibScormException.js"> </script>
	<script language="JavaScript"
		type="text/javascript" src="LMS.js"> </script>
	<script language="JavaScript"
		type="text/javascript" src="InterScoSeq.js"> </script>
	<script language="JavaScript" type="text/javascript">
	var lms = null;
	var iss = null;

	function load() {
		try {
			lms = new LMS(window);
			iss = new InterScoSeq(lms, function(){});
			if(iss.CanExitForward()) {
				document.getElementById("btnNext").disabled = false;
			}
			if(iss.CanExitBackward()) {
				document.getElementById("btnPrev").disabled = false;
			}
		} catch(e) {
			alert(e.toString());
		}
	}
	function unload() {
		if(lms && !lms.IsTerminated()) {
			lms.Terminate();
		}
	}
	</script>
</head>
<body onload='load()' onunload='unload()'>
	<input type="button" id="btnPrev" value="Previous SCO"
		disabled="true" onclick="iss.ExitBackward();" />
	<input type="button" id="btnNext" value="Next SCO"
		disabled="true" onclick="iss.ExitForward();" />
</body>
</html>
```

## 1.6 InterScoSeq Class Methods ##

Constructor(lms, onBeforeTerminate)
> The second argument is a function.
CanExitForward()
CanExitBackward()
CanExitChoice(target)
> Boolean.
ExitForward()
ExitBackward()
> The SCO will terminate.
ExitChoice(target)
> Available 'targets' are defined in content package sequencing.
ExitAll()
> As described in SCORM docs.

## 1.7 Decorators ##

Decorators allow you to conditionally add or change functionality in the library. The core library provides a decorator for InterScoSeq which is used like this:

```
lms = new LMS(window);
iss = new InterScoSeq(lms, function(){});
iss = NoExitBeforeComplete(iss);
```

This prevents `ExitForward()` and `ExitBackward()` from acting unless the SCO's `completion_status` is `completed`.

# 2 Multi-page SCO #

Often SCOs are broken into pages which share the same SCORM session. LibSCORM provides a class and decorators to implement this pattern.

## 2.2 Nav Class Methods ##

Constructor(pageUrls, contentDivId, lms, iss, asyncErrHandler, flashParams)
> The parameter `pageUrls` must be an array of either filenames or URLs. Both HTML and Flash SWFs are supported. HTML is loaded with AJAX, and SWFs are loaded with `swfobject`. The second parameter, `contentDivId`, is the name of a DOM identifier for a div in which the content will be loaded. The `lms` parameter -- and all parameters thereafter -- are optional. If `lms` is provided, it must be an instance of the LMS class. The Nav class can continue to operate even without `lms` but will not use SCORM features. The `asyncErrHanlder` must be a function. It is called if errors occur as a result of navigation events. If not specified, the default behavior will be to present a generic JavaScript alert box. Finally, `flashParams` (if supplied) must be an associative array of name-value pairs and is passed verbatim to the SWF. The constructor automatically calls `LoadLocation()`.
GotoPage(number)
> Loads the (zero-based) numbered element from the `pageUrls` array provided to the constructor. Returns `false` if the number is out of range.
NextPage(), PrevPage()
> Loads the next (or previous) page in the array. If already on the last (or first) page, it will use the `InterScoSeq` class to advance to the next (or previous) SCO in a content package if this is possible.
CurrentPageNum(), NumPages()
> What you would expect.
SaveLocation(), LoadLocation()
> Saves and loads the currently active page number as well as those of other pages which have been visited. Uses SCORM's `cmi.location` variable. If not running in an LMS, this function loads the first page, and marks the rest as unvisited.
GetVisitedRatio()
> Returns the number of visited pages divided by the number of total pages.
OnPageLoad, OnBeforeMove
> These are event hooks. Assign your functions to them to be notified.

# 3 Using the template #

[Watch](http://www.youtube.com/user/adljnelson) video tutorials.

To assemble a SCO from HTML or Flash files, copy the files into the `template` subdirectory of the LibSCORM distribution. Rename your HTML files as necessary before copying them to **avoid overwriting** the `index.html` file.

Next, edit `configure.js` in the `template` directory. This file controls some of your SCO's behavior. Add the names of the new file(s) to the NAV\_pages array. For instance

```
var NAV_pages = ['first.html', 'second.swf'];
```

If you save `configure.js` at this point and launch `index.html` from an LMS, your pages will appear with a navigation control to flip through them. Your SCO will automatically find the SCORM API and begin communication.

The file `configure.js` is self-documenting. See the examples inside it to learn its abilities.

## 3.1 For HTML ##

The template creates and provides the following global JavaScript variables to your content.
| **variable name** | **class** | **availability** |
|:------------------|:----------|:-----------------|
| dc                | DebugConsole | always available |
| lms               | LMS       | null if SCORM API not found |
| iss               | InterScoSeq | always available |
| nav               | Nav       | always available |
| objectives        | CMIBag    | null if lms is   |
| interactions      | CMIBag    | null if lms is   |

These variables simplify your code. For instance, here is a rewrite of example 1.1 which uses the `lms` object.

```
<html>
<body>
	<p id="message">Not connected to LMS</p>
</body>
<script language="JavaScript" type="text/javascript">
	if(lms) {
		document.getElementById("message").innerHTML =
			"Hello, " + lms.GetValue("cmi.learner_name");
	}
</script>
</html>
```

To test this, copy the above into a new HTML file, add the file to the `template` directory, and follow the steps in the previous section. When you launch it from an LMS, you should see it greet you. If you open `index.html` directly in a browser, it should say "Not connected to LMS."

## 3.2 For Flash ##

LibSCORM can display Flash SWF files as easily as it can HTML: simply add the Flash filename to `NAV_pages`. Flash SCOs often use their own flashy navigation and you can hide LibSCORM's default navigation bar by editing `configure.js`.

The easiest way to communicate with SCORM in Flash is through ExternalInterface. In Flash you can get the learner's name by calling

```
ExternalInterface.call("lms.GetValue", "cmi.learner_name");
```

Flash does not catch JavaScript exceptions, so be vigilant and check `lms.GetLastError()` frequently.