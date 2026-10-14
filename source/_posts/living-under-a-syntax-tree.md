---
title: Living in a Syntax Tree
date: 2026-10-14 17:32:40
tags:
---

Imagine I'm a token or a node in a syntax tree.

Maybe I'm an identifier, resting at a leaf;
Maybe I'm a semicolon, stand at the end of a statement;
Maybe I'm a keyword, I define the core rule of the language.
Or I'm a trivia token, meaningless but plays an important role in the lex stage. (split the tokens)

Well, why not be a node? Then I'll have many children.
I'll be produced from the event stream, or be build directly with a NodeBuilder.
If I am a node, I want to figure out how I produced.
Then I find the Sink, the Sink said you've created with the StartNode and FinishNode events.
I noticed the forward_parent field in the StartNode event variant, how did the sink produce me with just a Tombstone (event)? Then I see the trick: walking the chain of forward pointers, the Sink steps through the tombstones to wrap me from the inside out.
Do I have a parent? I might is a part of an expression;
I cannot imagine the whole world of the CST.
Can I be changed?
Can I be deleted?
No, I'm immutable.
So what's my value? If I'm missing, the program may still be compile. But with my existence, just influenced the semantic of the code.
So why I live? I think I live for the existence.
