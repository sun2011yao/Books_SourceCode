
Item 33: Use decltype on auto&& parameters to std::forward them.
One of the most exciting features of C++14 is  generic lambdas—lambdas that use
auto in their parameter specifications. The implementation of this feature is straight‐
forward: operator() in the lambda’s closure class is a template. Given this lambda,
for example,
```
auto f = [](auto x){ return func(normalize(x)); };
```
the closure class’s function call operator looks like this:
```
class SomeCompilerGeneratedClassName {
public:
  template<typename T>                   // see Item 3 for 
  auto operator()(T x) const             // auto return type
  { return func(normalize(x)); }
  …                                      // other closure class
};                                       // functionality
```
In this example, the only thing the lambda does with its parameter x is forward it to
normalize. If  normalize treats lvalues differently from rvalues, this lambda isn’t
written properly, because it always passes an lvalue (the parameter x) to normalize,
even if the argument that was passed to the lambda was an rvalue.
The correct way to write the lambda is to have it perfect-forward x  to normalize.
Doing that requires two changes to the code. First, x has to become a universal refer‐ence (see Item 24), and second, it has to be passed to normalize via std::forward
(see Item 25). In concept, these are trivial modifications:
```
auto f = [](auto&& x)
         { return func(normalize(std::forward<???>(x))); };
```
Between concept and realization, however, is the question of what type to pass to
std::forward, i.e., to determine what should go where I’ve written ??? above.
Normally, when you employ perfect forwarding, you’re in a template function taking
a type parameter  T, so you just write  std::forward<T>. In the generic lambda,
though, there’s no type parameter T available to you. There is a T in the templatized
operator() inside the closure class generated by the lambda, but it’s not possible to
refer to it from the lambda, so it does you no good.
Item 28 explains that if an lvalue argument is passed to a universal reference parame‐
ter, the type of that parameter becomes an lvalue reference. If an rvalue is passed, the
parameter becomes an rvalue reference. This means that in our lambda, we can
determine whether the argument passed was an lvalue or an rvalue by inspecting the
type of the parameter x. decltype gives us a way to do that (see Item 3). If an lvalue
was passed in,  decltype(x) will produce a type that’s an lvalue reference. If an
rvalue was passed, decltype(x) will produce an rvalue reference type.

Item 28 also explains that when calling std::forward, convention dictates that the
type argument be an lvalue reference to indicate an lvalue and a non-reference to
indicate an rvalue. In our lambda, if x is bound to an lvalue, decltype(x) will yield
an lvalue reference. That conforms to convention. However, if  x  is bound to an
rvalue,  decltype(x) will yield an rvalue reference instead of the customary non-
reference.
But look at the sample C++14 implementation for std::forward from Item 28:
```
template<typename T>                         // in namespace
T&& forward(remove_reference_t<T>& param)    // std
{
  return static_cast<T&&>(param);
}
```
If client code wants to perfect-forward an rvalue of type Widget, it normally instanti‐
ates  std::forward with the type  Widget (i.e, a non-reference type), and the
std::forward template yields this function:
```
Widget&& forward(Widget& param)            // instantiation of
{                                          // std::forward when
  return static_cast<Widget&&>(param);     // T is Widget
}

```
But consider what would happen if the client code wanted to perfect-forward the
same rvalue of type Widget, but instead of following the convention of specifying T to
be a non-reference type, it specified it to be an rvalue reference. That is, consider
what would happen if T were specified to be Widget&&. After initial instantiation of
std::forward and application of std::remove_reference_t, but before reference
collapsing (once again, see Item 28), std::forward would look like this:
```
Widget&& && forward(Widget& param)         // instantiation of
{                                          // std::forward when
  return static_cast<Widget&& &&>(param);  // T is Widget&&
}                                          // (before reference-
                                           // collapsing)
Applying the reference-collapsing rule that an rvalue reference to an rvalue reference
becomes a single rvalue reference, this instantiation emerges:
Widget&& forward(Widget& param)            // instantiation of
{                                          // std::forward when
  return static_cast<Widget&&>(param);     // T is Widget&&
}                                          // (after reference-
                                           // collapsing)
```
If you compare this instantiation with the one that results when  std::forward  is
called with T set to Widget, you’ll see that they’re identical. That means that instanti‐
ating std::forward with an rvalue reference type yields the same result as instantiat‐
ing it with a non-reference type.
That’s wonderful news, because decltype(x) yields an rvalue reference type when
an rvalue is passed as an argument to our lambda’s parameter  x. We established
above that when an lvalue is passed to our lambda, decltype(x) yields the custom‐
ary type to pass to std::forward, and now we realize that for rvalues, decltype(x)
yields a type to pass to std::forward that’s not conventional, but that nevertheless
yields the same outcome as the conventional type. So for both lvalues and rvalues,
passing  decltype(x) to  std::forward gives us the result we want. Our perfect-
forwarding lambda can therefore be written like this:
```
auto f =
  [](auto&& param)
  {
    return
      func(normalize(std::forward<decltype(param)>(param)));
  };
```
From there, it’s just a hop, skip, and six dots to a perfect-forwarding lambda that
accepts not just a single parameter, but any number of parameters, because C++14
lambdas can also be variadic:
```
auto f =
  [](auto&&... params)
  {
    return
    func(normalize(std::forward<decltype(params)>(params)...));
  };
```
## Things to Remember
• Use decltype on auto&& parameters to std::forward them.
