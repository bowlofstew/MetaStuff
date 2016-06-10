Meta stuff WIP
=======

This is the code I use in my game for all serialization/deserialization/introspection stuff.

Features
----

- **Strongly typed and doesn't use RTTI or virtual functions in any way**. You can iterate over class members and you still know member's type, there's no type erasure and it's all very fast
- **No dependencies**. You have to use modern C++ compiler which supports C++14, though. (VS 2015, GCC 5+, Clang 3.8)
- **Serialization is not limited to any format**. There's no standard way of doing serialization. You can implement it yourself for your own format. (See JSON example to see how it can be done)

The lib is still in development, so it's not recommended to use it for anything really serious as lots of stuff can change!
Still, use it as you like, it's MIT licensed after all.

All suggestions about improving the lib are welcome. I know it isn't perfect, so let's make it better together. :)

Requirements
----
- Compiler with C++14 support (I managed to compile it with Visual Studio 2015, GCC 5.0, Clang 3.8)

Dependencies
-----
- None! ([JsonCpp](https://github.com/open-source-parsers/jsoncpp) is used in example, but you can use any library you want for serialization)

Example
----

See example for complete example of JSON serialization.

Suppose you have classes like this:
```c++
struct Person {
    void setAge(int a)
    {
        if (a >= 0 && a < 128) { // sorry, if you're older than 128
            age = a;
        } else {
            std::cout << "Can't set age. " << a << " is out of allowed range\n";
        }
    }

    int getAge() const { return age; }

    int age;
    float salary;
    std::string name;
    std::unordered_map<std::string, std::vector<MovieInfo>> favouriteMovies;
};

struct MovieInfo {
    std::string name;
    float rating;
};
```
And you want to serialize them to some format (for example, JSON). Or perhaps you want to add some GUI which will let you edit each member easily.
No problem, just write these static functions,

```c++
#include "Meta.h"

namespace meta
{

template <>
inline auto registerMembers<Person>()
{
    return members(
        member("age")
            .addGetterSetter(&Person::getAge, &Person::setAge),
        member("salary", &Person::salary),
        member("name", &Person::name),
        member("favouriteMovies", &Person::favouriteMovies)
    );
}

template <>
inline auto registerMembers<MovieInfo>()
{
    return members(
        member("name", &MovieInfo::name),
        member("rating", &MovieInfo::rating)
    );
}

}
```
Note that you can either use pointers to members or pointers to getters/setters. They will be used for doing stuff with members of registered classes. (for reading and setting values).

and now you can call do this:
```c++
meta::doForAllMembers<SomeClass>(/* your lambda */);
```

Your lambda should have one parameter which will be an instance of Member. Calling ```meta::doForAllMembers<T>``` gives you ability to do something with each registered member of class T.
(See **example/JsonCast.inl** for examples of such lambdas).

Some docs (will be better in future!)
---

Member class has the following functions:

* `const char* getName()` - returns `const char*` of member name you've set during "registration"
* `const T& get(const Class& obj)` - gets const reference to the member
* `void set(const Class& obj, V&& value)` - sets value to the member, lvalues and rvalues are accepted
* `T& getRef(const Class& obj)` - gets non const reference to the member

If you provide Member with getters and setters it will use these functions for getting/setting members, otherwise the member will be accessed directly with pointer to member.

In general `Meta::getMembers<T>()` template function specialization should have a following form and should be put in header with you class (see comments in Meta.h for more info)

**It's important for this function to be `inline` and be defined in header file because the compiler has to figure out return type and you don't want to define the same function in two different compilation units!**

```c++
namespace meta
{

template <>
inline auto registerMembers<SomeClass>()
{
    return members(
        member(...),
        ...
    );
}

}
```

You can also add getters/setters and add non-const getter like this:

```c++
member("someMember", &SomeClass::someMember)
    .addGetterSetter(&SomeClass::getSomeMember, &SomeClass::setSomeMember)
    .addNonConstGetter(&SomeClass::getSomeMemberRef)
```

Getters and setters can be by-value:
```c++
// T is member type
T SomeClass::getSomeMember() const { return someMember; }
void SomeClass::getSomeMember(T value) { someMember = value; }
```

... or by reference
```c++
// T is member type
const T& SomeClass::getSomeMember() const { return someMember; }
void SomeClass::getSomeMember(const T& value) { someMember = value; }
```

Non-const getter has the following form:
```c++
// T is member type
T& SomeClass::getSomeMember() { return someMember; }
```

Getters and setters are always called (if they're present) in `Member::get/set` functions, otherwise the pointer to member is used. The same applies to non-const getter in Member::getRef.

You can make the template specialization of `registerMembers` a friend to your registered class to be able to access and add private members. You can do it like this:

```c++
class SomeClass {
    template <>
    friend auto registerMembers<SomeClass>(); // Visual Studio may produce warning here, just ignore it, it's a bug (`#pragma warning (disable : 4396)` is added in Meta.h)
};
```

License
---

This library is licensed under the MIT License, see LICENSE for more information.
