# YAML Basics

For Ansible, nearly every YAML file starts with a list. Each item in the list is a list of key/value pairs, commonly called a "hash" or a "dictionary". So, we need to know how to write lists and dictionaries in YAML.

There’s another small quirk to YAML. All YAML files (regardless of their association with Ansible or not) can optionally begin with **---** and end with **....** This is part of the YAML format and indicates the start and end of a document.

YAML uses spaces, period. Do not use tabs!! The suggested syntax for YAML files is to use **2 spaces for indentation**, but YAML will follow whatever indentation system that the **individual file uses**!

All members of a list are lines beginning at the same indentation level starting with a "**-** " (a dash and a space):

```

---
# A list of tasty fruits
fruits:
    - Apples
    - Oranges
    - Mango
...

```

A *dictionary* is represented in a simple key: value form (the colon must be followed by a space):


```

# An employee record
matt:
    name: Matt
    job: Developer
    skill: Godly

```

More complicated data structures are possible, such as **lists** of dictionaries, **dictionaries** whose values are lists or a mix of both:


```

# Employee records
-  matt:
    name: Matt
    job: Developer
    skills:
      - python
      - perl
      - pascal
-  james:
    name: James Lee
    job: Developer
    skills:
      - golang
      - c
      - perl

```

Dictionaries and lists can also be represented in an abbreviated form if you really want to:

```

---

matt: {name: Matt, job: Developer, skill: Godlike}
fruits: ['Apples', 'Oranges', 'Strawberries']

```


These are called "Flow collections".


Ansible doesn’t really use these too much, but you can also specify a boolean value (true/false) in several forms:

```

create_key: yes
needs_agent: no
knows_oop: True
likes_emacs: TRUE
uses_cvs: false

```

Values can span multiple lines using **|** or **>**. Spanning multiple lines using a "**Literal Block Scalar**" **|** will include the newlines and any trailing spaces. Using a "**Folded Block Scalar**" **>** will fold newlines to spaces; it’s used to make what would otherwise be a very long line easier to read and edit. In either case the indentation will be ignored. Examples are:

```

include_newlines: |
            exactly as you see
            will appear these three
            lines of poetry

fold_newlines: >
            this is really a
            single line of text
            despite appearances


```

While in the above **>** example all newlines are folded into spaces, there are **two** ways to enforce a newline to be kept:

```

fold_some_newlines: >
    a
    b

    c
    d
      e
    f
same_as: "a b\nc d\n  e\nf\n"

```

Let’s combine what we learned so far in an arbitrary YAML example. This really has nothing to do with Ansible, but will give you a feel for the format:


```

---
# An employee record
name: Msatt
job: Developer
skill: Godlike
employed: True
foods:
    - Apples
    - Oranges
    - Strawberries
languages:
    perl: Godly
    python: Godly
    pascal: Lame
education: |
    4 GCSEs
    3 A-Levels
    BSc in the Internet of Things

```


That’s all you really need to know about YAML to start writing Ansible playbooks.

# Gotchas


While you can put just about anything into an unquoted scalar, there are some exceptions. A colon followed by a space (or newline) :

```

`` is an indicator for a mapping. A space followed by the pound sign ``
# starts a comment.

```

Because of this, the following is going to result in a YAML syntax error:

```

foo: somebody said I should put a colon here: so I did

windows_drive: c:

```

...but this will work:

```

windows_path: c:\windows

```

You will want to quote hash values using colons followed by a space or the end of the line:

```

foo: 'somebody said I should put a colon here: so I did'


windows_drive: 'c:'

```

...and then the colon will be preserved.

Alternatively, you can use double quotes:

```

foo: "somebody said I should put a colon here: so I did"

windows_drive: "c:"

```

The difference between single quotes and double quotes is that in double quotes you can use escapes:

```

foo: "a \t TAB and a \n NEWLINE"

```

The following is invalid YAML:

```

foo: "an escaped \' single quote"

```

Further, Ansible uses **"{{ var }}"** for variables. If a value after a colon starts with a **"{"**, YAML will think it is a dictionary, so you must quote it, like so:

```

foo: "{{ variable }}"

```

If your value starts with a quote the **entire value must be quoted**, not just part of it. Here are some additional examples of how to properly quote things:

```

foo: "{{ variable }}/additional/string/literal"
foo2: "{{ variable }}\\backslashes\\are\\also\\special\\characters"
foo3: "even if it's just a string literal it must all be quoted"

```


Not valid:

```

foo: "E:\\path\\"rest\\of\\path\

```

In addition to *'* and *"* there are a number of characters that are special (or reserved) and cannot be used as the first character of an unquoted scalar:
```

[] {} > | * & ! % # ` @ ,

```

You should also be aware of ``? : -.`` In YAML, they are allowed at the beginning of a string if a non-space character follows, but YAML processor implementations differ, so it’s better to use quotes.

In Flow Collections, the rules are a bit more strict:

```
a scalar in block mapping: this } is [ all , valid

flow mapping: { key: "you { should [ use , quotes here" }

```

Boolean conversion is helpful, but this can be a problem when you want a literal yes or other boolean values as a string. In these cases just use quotes:

```
non_boolean: "yes"
other_string: "False"

```

YAML converts certain strings into floating-point values, such as the string 1.0. If you need to specify a version number (in a requirements.yml file, for example), you will need to quote the value if it looks like a floating-point value:

```

version: "1.0"

```


 [< Back: Basic Ansible Overview](https://github.com/sxcdennis/Ansible/blob/master/Basic.md) || [Next: Installing Ansible Automatically with Vagrant Boxes >](https://github.com/sxcdennis/Ansible/blob/master/vagrant.md)
