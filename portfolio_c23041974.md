# Task 1B

# Task 2B: Reflecting on the use of code review to improve code quality

During the initial stages of our project, the first thing we did was write testing and linting pipelines. This ensured that core functionality did not break and that code was uniformly formatted.
With this in place, obvious mistakes were quickly caught by the code owner before review. Standardized formatting to comply with PEP8 and prettier also minimized arguments on code style and formatting.

Despite the pipelines in place, code review is still essential to catch logic bugs, optimize code, and prevent security issues.

## Case 1: Caught a security bug
![Security bug caught](https://github.com/fidraC/team9-client-project/assets/144536228/23283328-9747-4047-a429-efe4a1352adf)

In this merge request, the decorators for checking if the user is an admin was refactored to be more generic, allowing checks for being logged in, is admin, or not logged in.
However, a mistake was made in the `if` condition `if ok and (check_admin or user.is_admin)` which would have allowed a non-admin user to access admin pages and API.

It was caught during code review and promptly fixed.
![image](https://github.com/fidraC/team9-client-project/assets/144536228/38fd1ea6-8231-41e3-8cd3-18ece31848ff)

## Case 2: Improve user experience

![Future proofing](https://github.com/fidraC/team9-client-project/assets/144536228/88cfcc84-5236-432e-aa79-3d3cc50bd4db)

In this scenario, if we were to add new error codes, it would not be caught and return a success even if the signup failed, confusing the user.
Looking back, this could've been better optimized using a dictionary instead of if conditions.
```py
code_to_error = {
  409: "User already exist",
  400: "Invalid user input",
  500: "Internal server error",
}
if status != 200:
  return redirect("/signup?error=" + code_to_error.get(status, f"Unknown error: {status}")
return redirect("/?message=Success")
```

## Case 3: Optimization

![DRY](https://github.com/fidraC/team9-client-project/assets/144536228/7cc8caea-8f2c-4256-a399-10c3cf73d68a)

While adding caching for the client's CSIL API, Uthman was able to recognize a sections of repetitive code. This allowed the removal of 19 lines of code and made the code more maintainable. Now, instead of having to modify 3 functions each time request details changed, only one had to be changed.

![image](https://github.com/fidraC/team9-client-project/assets/144536228/e4e4dd5c-0b26-47fb-8fa1-3744e28b3e55)

## Case 4: Minor efficiency gains

![image](https://github.com/fidraC/team9-client-project/assets/144536228/7aeb2978-e2a2-48a8-97cd-665fe1dce5e6)

> We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil.
>
> - Donald Knuth

Availability timeslots for demos can be specified by the admin. The UI allows them to fiddle around with it before submitting the entirety of the availability schedule as an array.
When adding that to the database, we currently loop one by one to add them to the database. This is of course less efficient than using a batch query.

However, there being only 168 hours a week, the maximum performance penalty is barely a few milliseconds. I therefore chose to temporarily ignore the suggestion so that we could focus on completing other features.
I believe there are times when not all optimizations are worth it. For example, during this time, the CSIL API cache has yet to been completed. When done, it reduced API latency from 700-900 ms to 200-300 ms. Compared to savings of less than 10 ms on an infrequent operation, it should objectively be put on the backlog and done in the future.
