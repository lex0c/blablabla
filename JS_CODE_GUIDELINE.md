# Code Guideline

If your code were flying to Mars, you wouldn’t want a memory leak to turn it into space junk. While you probably aren’t launching satellites with Node.js, these rules-adapted from [NASA’s Power of 10](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code) will keep your code safer, cleaner, and far less likely to detonate in production.

Follow these rules if you want your codebase to stay alive under heavy load, survive rookie mistakes, and be understood by someone who isn’t you six months from now.

1. **Avoid bizarre control flow structures**

   * No `goto` (yeah, it doesn’t even exist in JS, congrats), but also no infinite recursion or async hell with chained `.then().then().then()`.
   * Instead, use `async/await` and pure functions.

2. **All loops must have clear limits**

   * If the loop depends on external input (like a client request), enforce a hard limit (`maxIterations`) or a timeout.
   * No `while(true)` hoping it “will stop someday”.

3. **No uncontrolled dynamic allocation after initialization**

   * In C, this was `malloc` after setup; here it means avoiding huge structures created inside handlers for every request.
   * Preload configs, connections, and caches at startup.

4. **Functions should fit on one screen**

   * 50–60 lines max.
   * If you have to scroll, it’s time to break it down into smaller functions.
   * If you need to keep scrolling, your function is obese.

5. **Use sufficient assertions**

   * At least two serious validations per function.
   * `if (!param) throw new Error('...')`.
   * Validate not only the input but also the state before proceeding.

6. **Keep variable scope minimal**

   * Use `const` and `let` in the smallest possible block scope.
   * Global variables in Node.js are a disaster waiting to happen.
   * Avoid `var` like you avoid spoilers for a show.

7. **Always check return values**

   * Check the result of any function that returns a value or a Promise.
   * No `await doSomething()` and just pretending it worked.
   * Always handle `.catch()` or `try/catch`.

8. **Macros in JS? Don’t even think about it**

   * Here, that means no obscure metaprogramming, `eval`, or dynamic `Function()`.
   * Keep build/config simple.

9. **Limit pointer usage… or in this case, complex mutable references**

   * Don’t pass around massive mutable objects everywhere.
   * Prefer immutable copies (`{...obj}`) when possible.
   * Pure functions save lives (and debugging time).

10. **Compile (or analyze) with everything enabled and clean**

    * In Node.js, this means ESLint with no warnings and all tests passing.
    * Run `npm audit` and `npm outdated` like it’s a pre-launch checklist.
