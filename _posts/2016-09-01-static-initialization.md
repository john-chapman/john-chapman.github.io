---
layout: post
title: Ordering Static Initialization
date: 2016-09-01
comments: true
published: true
tags: c++, idioms, schwartz, nifty
---

In this post I'll discuss the problem of initializing global variables with `static` storage duration, and show you what I think is the best solution which C++ has to offer.

C++ makes some rather sparse guarantees about the initialization of such global variables:

- Storage for them is zero-initialized.
- Initialization happens either prior to the first statement of the main function, or before the first use of the variable.
- The order of initialization is guaranteed to happen in the same order as the variable declarations appear __within a single translation unit__. Initialization order _between_ translation units is not guaranteed.

That last point is what concerns us most: how can we order the initialization of `static` globals to reflect dependencies in our code?


Let's begin with an example which exhibits the behaviour we're talking about. Imagine we have a `Log` singleton which opens a file during initialization and provides a method to write to that file:

{% highlight cpp %}
// Log.h
#include <cstdio>

class Log {
	static Log s_instance;
	FILE* m_file;
	
	Log();
	~Log();
public:
	static Log* GetInstance() { return &s_instance; }
	void write(const char* _msg);
};
{% endhighlight %}

{% highlight cpp %}
// Log.cpp
#include "Log.h"

Log Log::s_instance;

void Log::write(const char* _msg) {
	fprintf(m_file, _msg);
}

Log::Log() {
	m_file = fopen("log.txt", "w");
}

Log::~Log() {
	fclose(m_file);
}
{% endhighlight %}

Note that we also close the file in the `Log` destructor, hence _deinitialization_ order is going to be important as well; we **cannot** call `Log::write()` before `Log::Log()` or after `Log::~Log()`.

Now let's imagine another singleton (imaginatively named `Subsystem`) which attempts to call `Log::write()` during initialization and deinitialization:

{% highlight cpp %}
// Subsystem.h
class Subsystem {
	static Subsystem s_instance;

	Subsystem();
	~Subsystem();
};
{% endhighlight %} 
{% highlight cpp %}
// Subsystem.cpp
#include "Subsystem.h"
#include "Log.h"

Subsystem Subsystem::s_instance;

Subsystem::Subsystem() {
	Log::GetInstance()->write("Subsystem Initialized!");
}

Subsystem::~Subsystem() {
	Log::GetInstance()->write("Subsystem Deinitialized!");
}
{% endhighlight %}

Obviously this relies on `Log::s_instance` being initialized **before** `Subsystem::s_instance`. If not, our calls to `Log::write()` will fail because the file was not created. Unfortunately there is no guarantee that this is the case. We are entirely at the whim of the compiler; `Log::s_instance` might be initialized before `Subsystem::s_instance` or vice versa. We just don't know.

_With Visual Studio I was able to get this code to fail consistently by ensuring that the `Subsystem` files appeared before the `Log` files inside the .vcxproj._

## Solution 1: Lazy Initialization ##
One solution is to use lazy initialization. During `Log::write()` we simply check if the file was created, and create it if it wasn't:

{% highlight cpp %}
void Log::write(const char* _msg) {
	if (!m_file) { // m_file is guaranteed to be zero-initialized
		m_file = fopen("log.txt", "w"); 
	}
	fprintf(m_file, _msg);
}
{% endhighlight %}

This is simple and it works. But what about deinitialization? The problem remains - it's still possible that `~Log()` will be called before `~Subsystem()`, which means that there is no sensible place to close our log file. It also means we have to test `m_file` every time we write to the log, which isn't ideal.

## Solution 2: Explicit Initialization/Deinitialization ##

Another idea might be to provide explicit `Init()` and `Shutdown()` methods for `Log` and `Subsystem` and move the ctor/dtor code into those methods. This way we can decide exactly when and where to create and destroy the log file:

{% highlight cpp %}
#include "Log.h"
#include "Subsystem.h"

int main(int, char**) {
	Log::Init();            // create the log file first
	Subsystem::Init();
	
	Subsystem::Shutdown();
	Log::Shutdown();        // destroy the log file last

	return 0;
}
{% endhighlight %}

This is a complete solution, but not a particularly elegant one. It depends entirely on the client code to order the dependencies, which becomes a maintenance nightmare as the number of `Init()`/`Shutdown()` pairs increases. Determining the order of dependencies is a hard problem for any non-trivial example, and what if the dependencies change and we forget to change the order of `Init()`/`Shutdown()` calls? Bad bad bad.

## Solution 3: Schwarz Counter ##

And so at last we come to the 'Schwarz' or 'Nifty' counter. The idea is simple: we declare a new class or struct `LogInit`, plus a static instance of `LogInit` _in the header_:

{% highlight cpp %}
// Log.h

class Log {
	friend class LogInit;
	static Log* s_instance;
	
  // remaining Log declaration as before
};

struct LogInit {
	LogInit();
	~LogInit();
	static int s_count;
};
static LogInit s_logInit;
{% endhighlight %}

This means that every translation unit which includes `Log.h` will get it's own static instance of `LogInit` and, therefore, `LogInit()` and `~LogInit()` will be called once for each translation unit. The initialization order is controlled by the counter `s_count`:

{% highlight cpp %}
// Log.cpp
#include "Log.h"

Log* Log::s_instance;
int LogInit::s_count;

LogInit::LogInit() {
	if (++s_count == 1) {
		Log::s_instance = new Log();
	}
}

LogInit::~LogInit() {
	if (--s_count == 0) {
		delete Log::s_instance;
	}
}

// remaining Log definition as before

{% endhighlight %}

_I've allocated `Log::s_instance` on the heap for brevity, but you can imagine other solutions e.g. using aligned storage and placement new, or calling static `Init()` and `Shutdown()` methods._

As you can see, we use `s_count` to ensure that the ctor/dtor are called exactly once, on the first and last calls to `LogInit()` and `~LogInit()` respectively. Because each translation unit gets its own `LogInit` instance, whichever 'goes first' during the pre-main initialization will increment `s_count` to 1 and subsequently call the `Log` ctor. Conversely, whichever translation unit 'goes last' during the post-main deinitialization will decrement `s_count` to 0 and call the `Log` dtor. Adding new dependencies 'just works' - we never have to worry about getting the initialization order right since the counter handles this for us. With one exception:

## Initialization Dependencies ##

What if `Log` depends on another singleton `FileSystem` (to create the log file), but `FileSystem` also depends on `Log` (to print an error message)? The Schwarz counter idiom can't help us here: there's no way to order the initialization of `Log` and `FileSystem` which works. 
The only advice in this case is to avoid cyclic dependencies like this - remove any `Log::GetInstance()->write()` calls from the `FileSystem` ctor/dtor, or vice versa. Dropping an `assert` in `GetInstance()` to check that the init happened is also useful. Hopefully these cases will be quite rare and easy to resolve by design. Note that this problem only applies to dependencies in the initialization/deinitialization - everywhere else is safe.


So that's the Schwarz counter - hopefully you can see why it's also called 'Nifty'.