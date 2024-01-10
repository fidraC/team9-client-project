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
I believe there are times when not all optimizations are worth it. For example, during this time, the CSIL API cache has yet to been completed. When done, it reduced API latency from 580-800 ms to 190-250 ms ms. Compared to savings of less than 10 ms on an infrequent operation, it should objectively be put on the backlog and done in the future.

# Task 3B: Discussion on the use of source control to improve software quality

Git can improve software quality and developer experience in a number of ways.
- By using `git bisect`, developers can easily find the exact commit in which a bug was introduced. As each git commit includes a snapshot of the repository at the time, we can always `reset --hard` to a working state. During our client project, the calendar picker mysteriously broke, causing an infinite loop and crashing browsers. Using `git bisect`, we were able to locate the commit causing it and selectively `git revert`.
- Branching allows different features to be developed in isolation and prevents having a moving target. For example, our back end code was factored into multiple modules, including the storage module that abstracted database access away from the core app. There were multiple times when the storage was refactored (e.g. to support multiple database formats). If this was done on the branch as another feature that used the module in a way no longer supported, more work would have to be done to keep up with changes. Instead with branching, this can be done instead: ![image](https://github.com/fidraC/team9-client-project/assets/144536228/8eb5d8fd-26da-4ae9-ba2d-804bcb176df2)

To keep everything even better versioned, we can use semantic versioning with git submodules. Using tagging, we can maintain multiple versions of the repository and backport universal changes that don't breaking compatibility via `git cherry-pick`.

Beyond `git` the version control system and towards GitLab specifically, we also made use of automated [workflows](https://git.cardiff.ac.uk/c22067305/team-9-client-project/-/blob/development/.gitlab-ci.yml?ref_type=heads) (GitLab CI/CD) for automated [testing](https://git.cardiff.ac.uk/c22067305/team-9-client-project/-/blob/development/test_storage.py?ref_type=heads) and [deployment](https://git.cardiff.ac.uk/c22067305/team-9-client-project/-/blob/development/deployment/app.py?ref_type=heads). The deployment is triggered when a merge request is made or when new commits are added to the associated branch, allowing us to quickly test functionality before merging. This also catches issues with dependency management (e.g. forgetting to put a package in `requirements.txt` which happened more than once) and generally ensures that the code doesn't *just* "works on my machine".

![image](https://github.com/fidraC/team9-client-project/assets/144536228/d1b6112f-3303-470e-be56-71d1e237b521)

Although we didn't get to that stage of the project, GitLab also offers error tracking. Via an SDK like sentry, you can catch both server side and client side errors and report then to GitLab. This is helpful in production to catch issues and its context when encountered by a user, improving response times to incidents. To ensure that our website stays online, we could also have used an uptime monitoring tool like [gatus](https://github.com/TwiN/gatus) and create a GitLab incident when service goes down for more than x minutes.

![image](https://github.com/fidraC/team9-client-project/assets/144536228/a3c3a271-8d86-40eb-a16c-55e7ff4421c1)

Overall, git has allowed us to ship more robust and reliable code while also improving our developer experience so that we can find/fix issues faster.
