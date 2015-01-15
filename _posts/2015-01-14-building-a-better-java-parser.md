---
layout: post
title:  "Building a Better Java Parser"
date:   2014-01-14 19:49
---
During my last post, I claimed all that was needed was to add the ability to my Transpiler to emit nested and anonymous classes, and by now I expected to be done with that. It turned out that what I though was a "good enough" Transpiler, was in fact, not.

In particular, its major failing was that its parsing was far too fragile. Even aside from trying to parse nested and anonymous classes, which should have been *relatively* easy to add, depending on the way the source Java was written, often enough it would incorrectly recognise a control structure, method call, etc and the parsing would fail.

Truthfully, my parser was pretty rubbish, but it was my first attempt at a parser and, unsurprisingly, I learnt many lessons along the way as I came to better understand the domain of parsing. While it's time consuming and disappointing that I had to resort to re-writing my parser, software developers learning such lessons in their inexperienced domain is actually pretty normal, hence why well experienced developers can normally demand a higher salary.

I have no formal training on how to write a parser and hence "winged it" at first, but having written one which mostly worked before, I know what worked well, and what I should change. My parser was broken into three parts:

1. [The Lexer](http://en.wikipedia.org/wiki/Lexical_analysis), which does tokenization of the input file(s)
2. The Parser, which uses the tokens to identify Java structures and then creates an [AST (Abstract Syntax Tree)](http://en.wikipedia.org/wiki/Abstract_syntax_tree).
3. The TypeScript Emitter, which traverses the AST and emits TypeScript code.

Lexing
---
To summarize lexing, it's the process of taking an input like:

{% highlight java %}
public class MyClass
{
	private int MyField;
	
	private void MyMethod(int par1, bool par2)
	{
		// Do something here
	}
}
{% endhighlight %}

And turning it into a list of tokens which would look like:

	[public]
	[class]
	[MyClass]
	[{]
	[private]
	[int]
	[MyField]
	[;]
	[private]
	[void]
	[MyMethod]
	[(]
	[int]
	[par1]
	[,]
	[bool]
	[par2]
	[)]
	[{]
	[// Do something here]
	[}]
	[}]

This process is pretty straightforward, it just takes a little time to get it right, and there are the occasional time consuming little gotcha's, like properly reading string literalls.

For my new transpiler, I am leaving the lexing almost completely untouched, but I have decided to "group" my tokens by types of brackets to make parsing easier, so my grouped tokens look like:

	[public]
	[class]
	[MyClass]
	<CURLY_BRACED_CONTENT_START>
		[private]
		[int]
		[MyField]
		[;]
		[private]
		[void]
		[MyMethod]
		<BRACED_CONTENT_START>
			[int]
			[par1]
			[,]
			[bool]
			[par2]
		<BRACED_CONTENT_END>
		<CURLY_BRACED_CONTENT_START>
			[// Do something here]
		<CURLY_BRACED_CONTENT_END>
	<CURLY_BRACED_CONTENT_END>
	
This makes my root list look as simple as:

	[class]
	[MyClass]
	<CURLY_BRACED_CONTENT>
	
This grouping is a very important change as it makes the next step of parsing, *much* easier.

Parsing
---
Next, parsing takes our tokens and turns them into an AST. The AST is just the simplist way of representing the code structures in a well organised [object graph](http://en.wikipedia.org/wiki/Object_graph). While in my transpiler, the object graph is composed of instances of C# classes, for demonstration purposes, a JSON object graph of an AST of the above could look like:

{% highlight js %}
JavaFile: {
	Class: {
		Name: "MyClass",
		Modifiers: ["public"],
		Fields: [
			{
				Name: "MyField",
				Modifiers: ["private"]
			}
		],
		Methods: [
			{
				Name: "MyMethod",
				Modifiers: ["private"],
				ReturnType: "void",
				Parameters: [
					{
						Name: "par1",
						Type: "int",
					},
					{
						Name: "par2",
						Type: "bool",
					}
				],
				Body: [
					{
						Type: "Comment",
						Content: "Do something here"
					}
				]
			}
		[
	}
}
{% endhighlight %}

Parsing was where I had a lot of problems in my old transpiler, it was very fragile due to the approach I was using to identify different code structures from my token input stream. My code looked something like this:

{% highlight c# %}
var bufferLength = 0;
do {
	if (IsMatchForMethodDeclaration(currentToken) {
		
		RewindTokenStream(bufferLength);
		ParseMethodDeclaration();
		continue;
	}
	
	if (IsMatchForFieldDeclaration) {
		...
	}
	
	...
	
	if (EndOfTokenStream())
	{
		break;
	}
	else
	{
		bufferLength++;
		currentToken = GetNextToken();
	}
} while (true);
{% endhighlight %}

To make matters worse, there only in a single "ParseBody" method, which was used everywhere, so this same "ParseBody" method was used to try identify class members and also expressions, like method invocations, with no concept of it's current context. This means that often enough it was difficult to distuinguish between method declarations and method invocations.

To make matters *even worse*, because a method declaration like `public void MethodName(bool par1, int par2, string par3) {...` was representented in the token stream like `[public], [void], [MethodName], [(], [bool], [par1], [,], [int], [par2], [,], [string], [par3], [)], [{]...`, recognizing it correctly using my "buffer" above was non-trivial because of the variable number of parameters.

What I found happening in practice is that it would incorrectly recoginize something, like a method invocation as a declaration, then start incorrectly going on a mission parsing incorrectly and failing for a reason of something like running out of elements when trying to find a closing brace, and only at that point would it throw the exception, when there was no easy way to see which element, half way earlier in the stream, was the actually mis-classified token.

Anwyay, like I mentioned at the start, in hindsight my original parser was pretty bad, it really was my [throw-away system](http://en.wikipedia.org/wiki/The_Mythical_Man-Month#The_pilot_system), the one I really should have expected would need to be made before the working system.

The TypeScript Emitter
---
This part is quite straightforward, once one has an AST, it's just a matter of deciding the syntax of your output and writing it. The only challenge was dealing with the devising TypeScript/JavaScript ways of representing some of the more complicated Java concepts. I have decided to improve my AST a little, so it will require largely re-writing the emitter, but I am not too worried about the amount of work involved in this part, as most of the work was deciding on the patterns of the output.

The Improved Parser
---

This meant that the part I really needed to improve was the parser. I already mentioned above that my first improvement was in the lexer where I grouped the tokens into braced content, which was one the essential components to help make the parsing easier.

Towards this end I have also created a simple "Token Input Stream" class, which is a little like an iterator, with methods like `.Next()`, `.HasNext()`, and `.Current()`. It's got a backing list of "Inputs", each input in the list is either a single input token or a group of input tokens, both types of which are stored in a private field in a list. It also has another private field which contains the current position of the list. Finally, I have methods which return a new instance TokenInputStream based on the current TokenInputStream:

- `GetTokenInputStreamForCurrentElement()` which will return an instance of TokenInputStream for the Input Tokens grouped by brace.
- `GetPeekStream()` which makes a new instance of TokenInputStream, but with the same backing list of Tokens and starting at the current position of the current stream. I use it to help identify which parser we should use, starting at the current input, based on a combination of upcoming elements. If there is a match, I throw away the PeekStream, and the parent stream is then used by the appropriate parser, not requiring it's position to be moved back or reset. Additionally, if there is no match, one can instantly see which element failed to be recognized.

Here is an example from trying to identify the members of a class:
{% highlight c# %}
var bodyTokenInputStream = TokenInputStream.GetTokenInputStreamForCurrentElement();

while (bodyTokenInputStream.HasNext())
{
	var currentInputElement = bodyTokenInputStream.Next();

	if (FieldParser.IsMatch(bodyTokenInputStream.GetPeekStream()))
	{
		var field = FieldParser.Parse(bodyTokenInputStream);
		ClassAstElement.FieldMembers.Add(field);
		continue;
	}

	if (MethodParser.IsMatch(bodyTokenInputStream.GetPeekStream()))
	{
		var method = MethodParser.Parse(bodyTokenInputStream);
		ClassAstElement.MethodMembers.Add(method);
		continue;
	}

	if (MemberClassParser.IsMatch(bodyTokenInputStream.GetPeekStream()))
	{
		var memberClass = MemberClassParser.Parse(bodyTokenInputStream);
		ClassAstElement.MemberClassMembers.Add(memberClass);
		continue;
	}

	if (StaticInitializerParser.IsMatch(bodyTokenInputStream.GetPeekStream()))
	{
		var staticInitializer = StaticInitializerParser.Parse(bodyTokenInputStream);
		ClassAstElement.StaticInitializer = staticInitializer;
		continue;
	}

	if (InstanceInitializerParser.IsMatch(bodyTokenInputStream.GetPeekStream()))
	{
		var instanceInitializer = InstanceInitializerParser.Parse(bodyTokenInputStream);
		ClassAstElement.InstanceInitializer = instanceInitializer;
		continue;
	}

	if (ConstructorParser.IsMatch(bodyTokenInputStream.GetPeekStream(), ClassAstElement as AbstractFullClassAstElement))
	{
		var constructor = ConstructorParser.Parse(bodyTokenInputStream);
		ClassAstElement.ConstructorMembers.Add(constructor);
		continue;
	}

	throw new Exception(string.Format("Unexpected token: {0}", currentInputElement));
}
{% endhighlight %}

The code is now very clean, and if it can't work out what the current element should be part of, it instantly fails on the correct element.

And here is an example of an `IsMatch()` method, used to detect if the current token is the start of a Method Declaration member of a class:

{% highlight c# %}
public static bool IsMatch(TokenInputStream tokenInputStream)
{
	var returnTypeFound = false;
	var methodNameFound = false;

	while (tokenInputStream.HasNext())
	{
		var currentElement = tokenInputStream.Next();
		var currentInputElementData = currentElement.InputElement != null
			? currentElement.InputElement.Data
			: null;

		if (!returnTypeFound && JavaUtils.IsMemberAccessModifier(currentInputElementData) || JavaUtils.IsMethodModifier(currentInputElementData))
		{
			continue;
		}

		if (!returnTypeFound && (currentInputElementData == Keywords.Void || JavaUtils.IsPrimitiveType(currentInputElementData) || currentElement.InputElement is IdentifierToken))
		{
			tokenInputStream.SkipOverPossiblyDotJoinedType();
			returnTypeFound = true;
			continue;
		}

		if (returnTypeFound &&
			!methodNameFound &&
			currentElement.InputElement is IdentifierToken &&
			tokenInputStream.Peek() != null &&
			tokenInputStream.Peek().InputType == InputType.BracedGrouped)
		{
			methodNameFound = true;
			tokenInputStream.Next();
			continue;
		}

		if (methodNameFound && (currentInputElementData == Keywords.Throws || currentElement.InputType == InputType.CurlyBracedGrouped))
		{
			return true;
		}

		return false;
	}

	return false;
}
{% endhighlight %}

I have opted for a *very* specific match and immediate return false if something doesn't match.

Using it in practice
---

So far I have only written the class parser, and it's working out *infinitely* easier compared to before. My new approach also allows me to validate outer structures, like classes parse, while skipping over innter structures (like expressions), allowing for much easier validation that my approach is working out.

With that, I'm signing off for this post. When I post the next one, I should be done with the parser.