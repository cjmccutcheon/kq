# kq

## Why

"kq" is a script I created because I was nesting too many kubectl commands with kubectl options, and my fingers were getting tired.

    kubectl logs --follow=true $(kubectl get -o yaml | grep 'gui' | head -n 1)

I believe that kubectl is something -- because it reports on so many fields on so many objects, and performs so many commands -- could really use a jq-like piping behavior.  Wouldn't this be better?

    kq :name pod /gui :first @follow

## The method

### Basics

Most kubectl actions are items whose output can be buffered and manipulated, like "kubectl get" and "kubectl events".  In the code, these are called "pcmd" for "pipe command", and their tokens start with ':'.

Some commands like "kubectl logs --follow" are foreground-only items, and essentially must terminate any action taken.  In the code, these are called "fcmd" for "foreground command", and their tokens start with '@'.  You should always find them at the end, if at all.

The pcmd functions all dump their output into a buffer and, if available, take the previous command's buffer as arguments to its own command.  (Tip: You can use "--input" to inject buffer content into the start of the command chain).

### Parameters to commands

Before going further, consider these quick demos:

    kq :name pod :grep gui :events :grep Live

This translates to "get all pods, isolate all 'gui' pods, and look for all events mentioning 'Live'"

Now consider:

    kq :name pod :grep -v gui :events :grep -i Live

This translates to "get all pods, remove all 'gui' pods, and look for all events mentioning 'live'"

You can see that flag options and bareword tokens all get passed to the "grep" executable as normal.

### Filters

I still do not want to be typing ":grep DEPLOYMENTNAME" too often, so there are two more tokens provided.

Tokens starting with '/' are passed to 'grep -Ei', and then run on the buffer just produced.  So ...

    kq :name pod /gui :events /live

is equivalent to ...

    kq :name pod :grep gui :events :grep Live

.

There is a second token type, prepended with '^'.  This is a negation, so it is sent to 'grep -Eiv'.  So ...

    kq :name pod ^gui :events /live

... is equivalent to 

    kq :name pod :grep -v gui :events :grep -i Live

Remember, special characters could still get mangled by your shell, so you may have to sometimes do:

    kq :name pod '/green[135]' :yaml

## In practice

Now that the concept is explained, you can understand that I have been using this command to shorthand things that are just a little too long in kubectl.

For example, YAML descriptors are very helpful.  So instead of saying:

    kq :name pod :get -o yaml

I can just create a _pcmd__yaml() function that runs "kubectl get -o yaml".  Now I have:

    kq :name pod :yaml

Some other quick commands I have added:

    - :json
    - :jsonpath
    - :name
    - :bash         -- Create a bash shell via "exec".  WARNING: Currently assumes you're in windows and need winpty.
    - :first        -- Choose the first item in a list (helpful for commands that need to take only one object)
    - :ordinal      -- Choose the N-th item in a list
    - :sed
    - :jq           -- Instead of :jsonpath, send :json to jq program (must be installed) and run inline command
    - :vi and :vim  -- Use vi or vim to edit the preceding buffer, and output the saved file as the next buffer. Exit with ":cq" to cancel.
    - :base64encode -- (Alias: :b64encode)
    - :base64decode -- (Aliases: :b64 and :b64decode)

### Selected Examples

#### Sample startTime from one of the pods named "foo"
    kq :name pod /foo :first :jsonpath '{.status.startTime}'

#### Decode secret "token" with name "sa-token"
    kq :name secret /sa-token :jsonpath '{.data.token}' :b64

#### Quick-and-dirty scan of "bar" image value from deployment "foo"
    kq :name deploy /foo :yaml /image: /bar

### Extending

If you do not want to edit kq directly, you can "export -f" your own _pcmd__foo() and _fcmd__bar() commands in your environment before calling kq.

### Miscellaneous

If you need to, you can change your namespace with "-n STRING" or "--namespace STRING".

If you need to start commands with text in the buffer, you can use "--input STRING".
