---
layout: post
title: Template Factory Pattern
date: 2016-11-26
comments: true
published: true
tags: c++, factory pattern, template
---

The **factory pattern** is one of the basic object-oriented design patterns, especially useful in cases where you want to instantiate classes based only on the class name or ID, without specifying the concrete type.

In this post I'll sketch out a general `Factory` template class, which can be used to implement the factory pattern with minimal modification of client code.

## Example ##

Let's start with an example where we'd like to apply the factory pattern. Say we have a `Projectile` interface and some deriving classes which implement different projectile behaviors:

{% highlight cpp %}
class Projectile
{
public:
	virtual void update() = 0;
};

class Bullet: public Projectile
{
public:
	virtual void update() { /* shoot */ }
};

class Grenade: public Projectile
{
public:
	virtual void update() { /* explode */ }
};
{% endhighlight %}

From this there are two main things it would be useful to do:

1. Instantiate `Bullet` or `Grenade` on demand using only the class name (e.g. for scripting purposes).
2. Iterate over all `Projectile` types and get the class name/ID (e.g. for a UI).

We'll return to (2) after I've presented the factory base class. For (1), we can imagine a `Create(const char* _name)` which might be implemented as follows:

{% highlight cpp %}
Projectile* Projectile::Create(const char* _name) 
{
	if (strcmp(_name, "Bullet") == 0) {
		return new Bullet;
	} else if (strcmp(_name, "Grenade") == 0) {
		return new Grenade;
	}
	return nullptr; // oops, name invalid
}
{% endhighlight %}

However it would be nicer to avoid writing a new `else if` clause or other boilerplate every time we add a new projectile.

_Incidentally, I prefer static `Create`/`Destroy` methods on the interface class over a separate `ProjectileFactory` class, if only to reduce namespace pollution. It doesn't really affect the way this implementation works._

## Factory Template	 ##

Below is the complete declaration of the `Factory` template class:

{% highlight cpp %}
template <typename T>
class Factory
{
public:

	struct ClassRef
	{
		typedef T*   (CreateFunc)();

		const char*  m_name;
		StringHash   m_nameHash;
		CreateFunc*  create;

		ClassRef(const char* _name, CreateFunc* _create)
			: m_name(_name)
			, m_nameHash(_name)
			, create(_create)
		{
			s_registry[m_nameHash] = this;
		}
	};

	template <typename U>
	struct ClassRefDefault: public ClassRef
	{
		static T* Create()
		{ 
			return new U; 
		}

		ClassRefDefault(const char* _name)
			: ClassRef(_name, Create)
		{
		}
	};

	static const ClassRef* FindClassRef(const char* _name);
	static int             GetClassRefCount();
	static const ClassRef* GetClassRef(int _i);

	static T*              Create(const char* _name)
	static T*              Create(const ClassRef* _cref);
	static void            Destroy(T*& _inst_);

private:
	static Map<const ClassRef*> s_registry;
};

#define FACTORY_DEFINE(_baseClass) \
	Map<const Factory<_baseClass>::ClassRef*> Factory<_baseClass>::s_registry
#define FACTORY_REGISTER(_baseClass, _subClass, _createFunc) \
	static Factory<_baseClass>::ClassRef s_ ## _subClass(#_subClass, _createFunc);
#define FACTORY_REGISTER_DEFAULT(_baseClass, _subClass) \
	static Factory<_baseClass>::ClassRefDefault<_subClass> s_ ## _subClass(#_subClass)
{% endhighlight %}

Key points:

- `Factory` maintains a container, `s_registry`, which maps class names (or name hashes) to instances of `ClassRef`. 
- `ClassRef` is metadata for the subclasses we want to instantiate (the class name/hash and a ptr to a create function).
- `ClassRefDefault` provides a default create function (which just calls `new`).
- `FACTORY_` macros are for convenience (since the template declarations are so ugly) and also to allow changes to the implementation without breaking existing code (e.g. changing the type of `s_registry`).
- Registration relies on static initialization to populate `s_registry`, hence care must be taken to avoid calling any of the methods of `Factory` during static initialization, because the initialization order [is not guaranteed](https://john-chapman.github.io/2016/09/01/static-initialization.html).
- `Destroy` takes a pointer _reference_ so that it can set `_inst_ = nullptr`, which just helps to catch dangling ptr bugs.

So, deriving from `Factory` we automatically get the required `Create` and `Destroy` methods which can instantiate any registered subclass by name. With this in mind, let's return to the `Projectile` example and 'factory-fy' it:

{% highlight cpp %}
// in the .h
#include "Factory.h"

class Projectile: public Factory<Projectile>
{
public:
	virtual void update() = 0;
};

// in the .cpp
FACTORY_DEFINE(Projectile); // define s_registry

class Bullet: public Projectile
{
public:
	virtual void update() { /* bullet stuff */ }
};
FACTORY_REGISTER_DEFAULT(Projectile, Bullet); // register Bullet

class Grenade: public Projectile
{
public:
	virtual void update() { /* grenade stuff */ }
};
FACTORY_REGISTER_DEFAULT(Projectile, Grenade); // register Grenade
{% endhighlight %}

_Note that the definitions of `Bullet` and `Grenade` don't need to be public - if client code is written only in terms of the `Projectile` interface they can be hidden away inside the .cpp, which incurs minimum recompilation when adding new projectile types (but see the notes about static libs below)._

## Listing Subclasses ##

The other desirable feature I mentioned earlier (iterating over registered subclasses) can be easily implemented in terms of `ClassRef`:

{% highlight cpp %}
for (int i = 0; i < Projectile::GetClassRefCount(); ++i) {
	const ClassRef* cref = Projectile::GetClassRef(i);
	// do stuff with cref, e.g. get the name or call Projectile::Create(cref)
}
{% endhighlight %}

## Static Libs ##

Using the code as I've presented it, you may run into issues when linking with a static library containing factory subclasses: the registration of the subclasses never happens! The reason is this: because the linker is free to ignore any code in the static lib which it decides has no effect on the executable, the static variables declared by the `FACTORY_REGISTER` macros are ignored.

One solution to this problem is to tell the linker not to do this: use `/WHOLEARCHIVE` for MSVC or `--whole-archive` for GCC. This forces the linker to include everything: it is an all-or-nothing option, though, which can bloat the final executable (though in practice not by much).

We could also exploit the fact that if _any_ code in the same translation unit as the `FACTORY_REGISTER` macros is referenced by the executable, the linker must include the whole translation unit. In the example above this is already the case; `s_registry` is defined in the same .cpp that our `FACTORY_REGISTER` macros appear and because it is externally visible (via the `Projectile` interface) the linker has to link everything in the file. If this were not the case, for example if we defined `Bullet` and `Grenade` in a separate .cpp file, we could use a little hack to achieve the same result:

{% highlight cpp %}
// helper macros
#define FORCE_LINK_REF(_name) int _forceLinkRef_ ## _name;
#define FORCE_LINK(_name) \
	void _forceLink_ ## _name(void) { \
		extern int _forceLinkRef_ ## _name; \
		++_forceLinkRef_ ## _name; \
	}
	
// in the .cpp in the static lib
FORCE_LINK_REF(Projectile);

// in a .cpp in the .exe linking the static lib
FORCE_LINK(Projectile);
{% endhighlight %}

This is ugly, but is the most portable solution. The `FORCE_LINK_REF` macro declares an `int` in the translation unit we want to force-link and references it from a function in the executable, defined via the `FORCE_LINK` macro. Note that it's not necessary to actually call the function, simply defining it does the trick.

## Initialization Order ##

Another problem arises in that initialization order of `s_registry` and the static `ClassRef` instances (which add themselves to the registry) are not guaranteed. Depending on the choice of container for `s_registry` this may break the whole system - if `ClassRef` instances register themselves before `s_registry` is initialized it may either crash, or overwrite some or all of the registered pointers when the container is initialized. Bad bad bad.

Fortunately the solution is simple - by amending the code in the `ClassRef` ctor we can init the container on demand:

{% highlight cpp %}
ClassRef(const char* _name, CreateFunc* _create)
	: m_name(_name)
	, m_nameHash(_name)
	, create(_create)
{
	if (!s_registry) {
		s_registry = new Map<ClassRef*>; // s_registry is now a ptr
	}
	(*s_registry)[m_nameHash] = this;
}
{% endhighlight %}

## Conclusion ##

So that's it - mostly basic stuff, but I didn't find an implementation exactly like this when looking around. Of course it could be extended to add more features:

- A custom `Destroy` function per subclass, although this is probably of limited use since you'll usually have virtual destructors.
- Add a `ClassRef*` member to `Factory`; this way all subclasses have access to their class metadata which would be a simple way of doing RTTI.

You can see a complete implementation in my [ApplicationTools](https://github.com/john-chapman/ApplicationTools) project [here](https://raw.githubusercontent.com/john-chapman/ApplicationTools/master/src/all/apt/Factory.h). 
