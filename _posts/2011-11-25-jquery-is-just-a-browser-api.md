---
layout: post
title: jQuery is just a browser API
tags:
- javascript
- jquery
status: publish
type: post
published: true
meta:
  _edit_last: "1"
---

[Update] Thanks to Charlie in the comments for pointing me to jQuery's <code>$.fn.proxy</code> method.

Developers have traditionally used JavaScript for relatively simple DOM manipulations and XHR, but as more functionality moves to the client, the techniques used by those developers have been slow to evolve. One consequence of this slow evolution is systems comprised entirely of jQuery and a series of nested event bindings. While jQuery is an extremely effective browser API, hiding the clumsy and often bug ridden out of the box equivelants, developers need effective solutions for organizing and reusing their code.

<h2>Nested Closures</h2>

Relegating code to event bindings breaks down in even very simple situations for a few important reasons. First the state that's closed over is often hard to track and bindings can easily be shadowed.

<pre>
(<span class="keyword">function</span>( <span class="js2-function-param">$</span> ) {
  <span class="keyword">function</span> <span class="function-name">other</span>() {
    <span class="comment">// ...
</span>  }

  <span class="keyword">function</span> <span class="function-name">header</span>() {
    <span class="comment">//...
</span>  }

</span>
  $(<span class="string">".bar"</span>).click(<span class="keyword">function</span>() {
    <span class="keyword">var</span> <span class="variable-name">header</span> = $( <span class="string">"header"</span> );

    <span class="comment">// ...
</span>  });
})( jQuery );</pre>

Developers altering the <code>click</code> callback are required to know all the bindings in the current and parent scopes or face the possibility of attempting something unfortunate like using the invocation operator on a jQuery object. Second the inclusion of the state in each callback is implicit which means everything in the enclosing scope is carried with the callback. This is the beauty of the closure when used properly, but here it only adds complexity and possibly memory bloat. Third, and most importantly, the code tied up in the functions and callbacks is entirely unusable outside that context.

<h2>Objects Not Functions</h2>

As far as alternatives go, it is possible to write JavaScript in a functional style. Sadly JavaScript lacks many language features that make functional programming easy and elegant. Partial application, pattern matching, tail call optimization, and simple composition are all missing. It's also a style of programming that is unfamiliar to the vast majority of front end developers. For now objects appear to be the best means of code reuse and organization where JavaScript is concerned.

With that in mind there are a number of options worth considering. Many JavaScript libraries provide a class system to emulate similar systems in other languages. <a href="http://mootools.net/" title="MooTools">MooTools</a> is a prime example in that it's <a href="http://mootools.net/docs/core/Class/Class" title="class system">Class</a> constructor supports classical inheritance, mixins, and more. This approach has the obvious benefit of being relatively familiar to developers without a lot of JavaScript experience and if you prefer to work with the DOM via jQuery you can even discard the MooTools DOM API with the download builder.

Additionally a lot of client side software involves state management between JavaScript objects and some DOM representation. Libraries like <a href="http://documentcloud.github.com/backbone/#Model" title="Backbone Model">Backbone</a> and <a href="http://docs.sproutcore.com/#doc=SC.Object&src=false" title="Sproutcore Object">Sproutcore</a> provide models in one form or another that can act as a class replacement. As Yehuda Katz pointed out in his <a href="http://vimeo.com/22687694" title="Getting Truth from the DOM">presentation</a> "Getting Truth Out of the DOM", attempting to manage complex state between the the DOM and JavaScript is best left to an existing solution. Taken with the need to better organize an application these libraries can be invaluable.

Finally, you can use the (gasp) prototype system provided by JavaScript. The aversion to JavaScript's object model appears to be an unfortunate side effect of the overall lack of experience with the language, but by creating a constructor and then defining methods on it's prototype you can group, organize, and reuse common functionality quite easily<sup>1</sup>. Even when choosing another solution, neglecting to understand JavaScript at this level is an unfortunate misstep.

Operating under the assumption that jQuery will remain the primary means of interacting with the DOM for most developers it's necessary to address binding context (ie <code>this</code>) as it becomes necessary when using object methods as arguments to higher order functions (eg event handlers).

<h2>jQuery and Function.prototype.bind</h2>

<del>One conspicuous absence from jQuery is a helper for function context binding</del> jQuery's answer for context binding is the <code>$.proxy</code> method. That is, it provides a way to guarantee the value of <code>this</code> during the invocation of a given function. <code>this</code> becomes important very quickly in even simple attempts to reconcile DOM events with code organization through objects. Here we'll build a library agnostic version.

<pre>
<span class="keyword">function</span> <span class="function-name">AlertArea</span>( <span class="js2-function-param">$domElement</span> ){
  <span class="comment">// bind to the instances onCloseClick
</span>  $domElement.find( <span class="string">"div.close"</span> ).click(<span class="builtin">this</span>.onCloseClick);

  <span class="comment">// if the alert area has text on creation show it
</span>  <span class="keyword">if</span>( $domElement.text() ) {
    $domElement.show();
  }

  <span class="comment">// track the dom element for
</span>  <span class="builtin">this</span>.$domElement = $domElement;
}

AlertArea.prototype.<span class="function-name">onCloseClick</span> = <span class="keyword">function</span>() {
  <span class="comment">// BOOM!!!
</span>  <span class="builtin">this</span>.$domElement.hide();
};</pre>

In this example when the constructor is invoked with some jQuery wrapped DOM object, if that object has text, it's displayed. It also binds the prototype method <code>onCloseClick</code> to clicks on child elements matching the <code>div.close</code> selector. The problem here is that <code>this</code> in <code>onCloseClick</code> won't be the <code>AlertArea</code> object when the click event is triggered but rather the DOM element on which the event was triggered (a feature of jQuery). To guarantee the <code>this</code> value upon invocation a binding function is required.

<pre>
<span class="keyword">function</span> <span class="function-name">bindContext</span>(<span class="js2-function-param">newThis</span>, <span class="js2-function-param">fn</span>) {
  <span class="keyword">return</span> <span class="keyword">function</span>() {
    fn.apply(newThis, arguments);
  };
}</pre>

This function takes as arguments a context (<code>newThis</code>) and a function (<code>fn</code>). It then creates a new function that, when invoked, will use the <code>apply</code> method of the original function to invoke it with the new context. More commonly this is defined on the function prototype.

    <pre>
Function.prototype.<span class="function-name">bind</span> = <span class="keyword">function</span>(<span class="js2-function-param">newThis</span>) {
  <span class="keyword">var</span> <span class="variable-name">self</span> = <span class="builtin">this</span>;

  <span class="keyword">return</span> <span class="keyword">function</span>() {
    self.apply(newThis, arguments);
  };
};</pre>

With that in place the <code>this</code> value of the <code>onCloseClick</code> method can be guaranteed by altering the click binding

<pre>
<span class="comment">// bind to the instances onCloseClick
</span>
$domElement.find( <span class="string">"a.close"</span> ).click(<span class="builtin">this</span>.onCloseClick.bind(<span class="builtin">this</span>));
</pre>

<del>The fact that jQuery lacks this core function is telling. It isn't attempting to facilitate your code organization because that's not what it's for. I like to think that at least part of jQuery's success is due to it's focus on a very specific set of problems, but that leaves the developer to choose their preferred method of code organization. Sadly many developers fail to take that step though it can easily be accomplished with JavaScript's prototype system</del>.

<h2>Getting Comfortable With <code>this</code></h2>

A simple library for managing textareas should serve to illustrate how the prototype system is an easy win over nested closures both early on and as application complexity grows. We'll start with the more common approach:

<pre>
$( <span class="string">"textarea"</span> ).each(<span class="keyword">function</span>(<span class="js2-function-param">i</span>, <span class="js2-function-param">elem</span>) {
  <span class="keyword">var</span> <span class="variable-name">$textArea</span> = $(elem);

  <span class="keyword">function</span> <span class="function-name">grow</span>(){
    <span class="keyword">var</span> <span class="variable-name">scrollHeight</span> = $textArea[ 0 ].scrollHeight,
        <span class="variable-name">clientHeight</span> = $textArea[ 0 ].clientHeight;

    <span class="keyword">if</span> ( clientHeight &lt; scrollHeight ) {
      $textArea.height(scrollHeight + 20);
    }
  }

  <span class="keyword">function</span> <span class="function-name">limitWarn</span>() {
    <span class="keyword">if</span>( $textArea.val().length &gt; 10 ) {
      $textArea.css( <span class="string">'background-color'</span>, <span class="string">'red'</span> );
    }
  }

  $textArea.keyup(grow).keyup(limitWarn);
  $(window).load(grow).load(limitWarn);
});
</pre>

For all textarea elements in the DOM the <code>grow</code> and <code>limitWarn</code> functions are bound to <code>keyup</code> for user input and <code>$(window).load</code> for text included in the markup. <code>grow</code> does a simple calculation to determine if the text exceeds the textarea's rendered height and then increases it where necessary. <code>limitWarn</code> changes the background color of the textarea when the content length exceeds a given threshold. Already there are three references bleeding into <code>grow</code> and <code>limitWarn</code> (ie, <code>i</code>, <code>elem</code>, and <code>limitWarn</code>/<code>grow</code>). Now, the equivalent using the prototype system.

<pre>
<span class="keyword">function</span> <span class="function-name">TextArea</span>( <span class="js2-function-param">$textArea</span>, <span class="js2-function-param">limit</span> ){
  <span class="keyword">var</span> <span class="variable-name">grow</span> = <span class="builtin">this</span>.grow.bind(<span class="builtin">this</span>),
      <span class="variable-name">limitWarn</span> = <span class="builtin">this</span>.limitWarn.bind(<span class="builtin">this</span>);

  $textArea.keyup(grow).keyup(limitWarn);
  $(window).load(grow).load(limitWarn);

  <span class="builtin">this</span>.$textArea = $textArea;
  <span class="builtin">this</span>.limit = limit;
}

TextArea.prototype.<span class="function-name">grow</span> = <span class="keyword">function</span>() {
  <span class="keyword">var</span> <span class="variable-name">scrollHeight</span> = <span class="builtin">this</span>.$textArea[ 0 ].scrollHeight,
      <span class="variable-name">clientHeight</span> = <span class="builtin">this</span>.$textArea[ 0 ].clientHeight;

  <span class="keyword">if</span> ( clientHeight &lt; scrollHeight ) {
    <span class="builtin">this</span>.$textArea.height(scrollHeight + 20);
  }
};

TextArea.prototype.<span class="function-name">limitWarn</span> = <span class="keyword">function</span>() {
  <span class="keyword">if</span>( <span class="builtin">this</span>.$textArea.val().length &gt; <span class="builtin">this</span>.limit ) {
    <span class="builtin">this</span>.$textArea.css( <span class="string">'background-color'</span>, <span class="string">'red'</span> );
  }
};

$(<span class="string">"textarea"</span>).each(<span class="keyword">function</span>(<span class="js2-function-param">i</span>, <span class="js2-function-param">elem</span>) {
  <span class="keyword">new</span> TextArea( $(elem), 10 );
});
</pre>

The only additional overhead is the function context bindings<sup>2</sup>, typing <code>TextArea</code> and <code>prototype</code>, and the instantiation. The upshot is a purpose built constructor where the <code>each</code> closure served before and a very specific portal through which state contained in the object is accessed (<code>this</code>).

Now consider another developer working on the same project needs to manage textareas in a different context with a minor alteration. The closure based solution requires something of an overhaul, but the <code>TextArea</code> prototype can remain nearly untouched while it's repurposed elsewhere.

    <pre>
<span class="keyword">function</span> <span class="function-name">DialogWarnTextArea</span>( <span class="js2-function-param">$textArea</span>, <span class="js2-function-param">limit</span>, <span class="js2-function-param">$dialog</span> ) {
  TextArea.apply(<span class="builtin">this</span>, arguments);

  <span class="builtin">this</span>.$dialog = $dialog;
}

</span>DialogWarnTextArea.prototype = <span class="keyword">new</span> TextArea();

</span>DialogWarnTextArea.prototype.<span class="function-name">limitWarn</span> = <span class="keyword">function</span>() {
  <span class="keyword">if</span>( <span class="builtin">this</span>.$textArea.val().length &gt; <span class="builtin">this</span>.limit ) {
    <span class="builtin">this</span>.$dialog.text(<span class="string">"Too much text!"</span>).show();
  }
};</pre>

A few things to take note of. First, an application of the original <code>TextArea</code> constructor to <code>this</code> in the <code>DialogWarnTextArea</code> constructor makes sure the original attributes are set up properly (ie <code>limit</code> and <code>$textArea</code>). You can think of this as an explicit <code>super</code>.

<pre>
</span>TextArea.apply(<span class="builtin">this</span>, arguments);
</pre>

Second, assigning the prototype to a new instance of <code>TextArea</code> places the methods and attributes defined on it's prototype in the chain above those defined (later) on the <code>DialogWarnTextArea</code> prototype. This is referred to by most JavaScript developer's as "prototypal inheritance" for the way it emulates method/attribute lookup in languages with classical inheritance<sup>3</sup>.

<pre>
</span>DialogWarnTextArea.prototype = <span class="keyword">new</span> TextArea();
</pre>

Third, defining a <code>limitWarn</code> attribute on the <code>DialogWarnTextArea</code> prototype means that objects created with the <code>DialogWarnTextArea</code> will use that definition since it appears earlier in the prototype chain.

    <pre>
DialogWarnTextArea.prototype.<span class="function-name">limitWarn</span> = <span class="keyword">function</span>() {
  <span class="keyword">if</span>( <span class="builtin">this</span>.$textArea.val().length &gt; <span class="builtin">this</span>.limit ) {
    <span class="builtin">this</span>.$dialog.text(<span class="string">"Too much text!"</span>).show();
  }
};
</pre>

Last, we need to handle the case where the <code>TextArea</code> constructor is invoked with the new operator to set a prototype (ie, without arguments) which constitutes our only alteration to the <code>TextArea</code> code.

<pre>
<span class="keyword">function</span> <span class="function-name">TextArea</span>( <span class="js2-function-param">$textArea</span>, <span class="js2-function-param">limit</span> ){
  <span class="keyword">if</span>( !arguments.length ) <span class="keyword">return</span>;
  <span class="comment">// ...
</span>}
</pre>

It should also be pointed out that there are a few items worth refactoring (3 arg and the <code>limitWarn</code> boolean operation) but the changes are at least palatable and mostly non-invasive. In stark contrast, achieving the same functionality without the prototype system requires moving the <code>grow</code> function into a parent scope, gathering the target from the event object passed to <code>grow</code> as an event handler, and creating another each loop over the second set of textareas.

<pre>
<span class="keyword">function</span> <span class="function-name">grow</span>( <span class="js2-function-param">event</span> ){
  <span class="keyword">var</span> <span class="variable-name">$textArea</span> = $(event.target),
      <span class="variable-name">scrollHeight</span> = $textArea[ 0 ].scrollHeight,
      <span class="variable-name">clientHeight</span> = $textArea[ 0 ].clientHeight;

  <span class="keyword">if</span> ( clientHeight &lt; scrollHeight ) {
    $textArea.height(scrollHeight + 20);
  }
}

$( <span class="string">"textarea.other"</span> ).each(<span class="keyword">function</span>( <span class="js2-function-param">i</span>, <span class="js2-function-param">elem</span> ) {
  <span class="keyword">var</span> <span class="variable-name">$textArea</span> = $(elem);

  <span class="keyword">function</span> <span class="function-name">limitWarn</span>() {
    <span class="keyword">if</span>( $textArea.val().length &gt; 10 ) {
      $(<span class="string">"#dialog"</span>).text(<span class="string">"Too much text!"</span>).show();
    }
  }

  $textArea.keyup(grow).keyup(limitWarn);
  $(window).load(grow).load(limitWarn);
});</pre>

Also unfortunate is that <code>grow</code> has lost context for the reader, it relies entirely on the fact that the <code>event.target</code> will behave like a textarea. Ultimately this is really about getting any reusability <strong>at all</strong> with the added bonuses of sane state access and a limit on the enclosed bindings. Using closures alone to manage scoping concerns simply collapses where reuse is concerned.

<h2>Slow Progress</h2>

My fear, and the motivation for this post, is that developers are not turning quickly enough to libraries and language constructs, be they MV* frameworks or plain old prototypes, which are crucial to building more complex client side applications. For nearly any application that must be maintained, even those with a minimum of client side code, developer's should look to move beyond the simple scoping semantics of the jQuery event callback and leverage the code organization and reuse features that JavaScript and its libraries have to offer.

<ol>
	<li>One of the best examples I've found is Google's closure library. Whatever you may think of the library itself, the <a href="http://code.google.com/p/closure-library/source/browse/trunk/closure/goog/demos/samplecomponent.js" title="Sample">code</a> is really nice to look through.
	<li>Underscore.js provides a nice facility for binding all methods on an object to that object context (<code>_.bindAll</code>), though it requires explicit inclusion of methods defined on the prototype</li>
	<li>You can learn more about prototypal inheritance from <a href="http://javascript.crockford.com/prototypal.html" title="prototypal inheritance">Douglas Crockford</a> and <a href="http://yehudakatz.com/2011/08/12/understanding-prototypes-in-javascript/" title="Yehuda Katz - Prototypes">Yehuda Katz</a></li>
</ol>


