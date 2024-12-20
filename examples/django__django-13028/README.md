## Introduction

This is the django\_\_django-13028 task taken from the [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) dataset. See also:

* Original issue: https://code.djangoproject.com/ticket/31664

* Fixing PR: https://github.com/django/django/pull/13028

## Takeaway

When dealing with a software issue, the LLM fails to follow project-specific conventions, leading to an incorrect bug fix.


## Input and Output

* Input: [input.json](input.json). This is a conversation between our tool and an LLM that analyzes the issue and finds the correct patching location.

* Actual Output: [output.md](output.md). This contains an LLM-generated patch for the issue. The patch can be reformatted to the diff format, see [output.diff](output.diff).

* Expected Output: [expected_output.diff](expected_output.diff). This is a patch written by a human expert.

For ease of reading, we have also included the [original issue statement](issue_statement.md).

## Detailed Analysis

In this example, the LLM has diagnosed the issue accurately, as in [output.md](output.md):

> The current implementation checks for the presence of a `filterable` attribute on the expression and raises an error if it's False. This is causing problems when a model has a field named 'filterable'.

It also points out the correct fix:

> To achieve this, we'll change the condition to check if the expression is an instance of Expression (from django.db.models.expressions) and if it has a filterable attribute set to False.

However, the specific check of `Expression` generated by the LLM is incorrect. It is a static type check:

```diff
     def check_filterable(self, expression):
         """Raise an error if expression cannot be used in a WHERE clause."""
-        if not getattr(expression, 'filterable', True):
+        from django.db.models.expressions import Expression
+        if isinstance(expression, Expression) and getattr(expression, 'filterable', True) is False:
             raise NotSupportedError(
                 expression.__class__.__name__ + ' is disallowed in the filter '
                 'clause.'
```

Whereas the human expert performed a duck-typing check:

```diff
     def check_filterable(self, expression):
         """Raise an error if expression cannot be used in a WHERE clause."""
-        if not getattr(expression, 'filterable', True):
+        if (
+            hasattr(expression, 'resolve_expression') and
+            not getattr(expression, 'filterable', True)
+        ):
             raise NotSupportedError(
                 expression.__class__.__name__ + ' is disallowed in the filter '
                 'clause.'
```

Arguably, this duck-typing check (`hasattr(expression, 'resolve_expression')`) is
better than the LLM-generated static type check, as duck typing gives more flexibility.
Moreover, exactly the same check of `resolve_expression` occurs at multiple other
places in the codebase as well as in the [documentation](https://docs.djangoproject.com/en/3.0/ref/models/expressions/#writing-your-own-query-expressions). Therefore, the duck-typing
check is more in line with the project-specific conventions and would be more acceptable
to the project maintainers.

Generally, if an LLM-generated bug fix does not follow project-specific conventions, it
would require more human intervention to ensure that the fix is acceptable to the project.
More importantly, such fixes are more often incorrect. Failure to follow project-specific
conventions limits the usefulness and trustworthiness of LLMs in real-world software maintenance.
