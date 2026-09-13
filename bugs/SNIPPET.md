- Error rendering template:
⨯ Error: Functions cannot be passed directly to Client Components unless you explicitly expose it by marking it with "use server". Or maybe you meant to call this function rather than return it.
  {lang: "rust", name: ..., children: </>, preview: ..., archetype: ..., type: ..., fileType: function j$, files: ..., service: ..., database: ...}
                                                                                              ^^^^^^^^^^^
    at ignore-listed frames {
  digest: '519587519'
}

Template:

model: template
metadata:
  name: nadl_01M1A1YCQ4XHR18QER8Q3B14Z5
  namespace: lingv
  uid: uadl_01M1A1YCQ4T6CSEHDQB42VYD2H
labels:
  - culture: en-US
    title: Ownership and Borrowing
    overview: >-
      This lesson introduces Rust's ownership model, the rules of ownership, and
      the concepts of borrowing and references. You will learn how Rust
      guarantees memory safety without a garbage collector and how to write code
      that correctly manages resources.
    tooltip: Ownership and Borrowing
    tags:
      - rust
      - ownership
      - borrowing
      - references
def:
  type: lesson
  format: markdown
  content:
    - culture: en-US
      body: >
        # Ownership and Borrowing


        ## Introduction


        Rust's ownership system is the language's core mechanism for ensuring
        memory safety without a garbage collector. The compiler enforces a set
        of rules that determine when memory is allocated and freed.
        Understanding these rules is essential for writing safe and idiomatic
        Rust code.


        ## Ownership Rules


        The ownership model rests on three rules:


        1. Every value in Rust has a single owner.

        2. There can be only one owner at a time.

        3. When the owner goes out of scope, the value is dropped.


        These rules are enforced at compile time. If you violate them, the code
        does not compile.


        The following diagram shows how ownership transfers when a value is
        moved:


        <Diagram type='mermaid' name="uadl_01M1A1YCQ4T6CSEHDQB42VYD2H_00">{`

        flowchart TD
            A["Variable owns value"] --> B["Move to another variable"]
            B --> C["Original variable no longer valid"]
            B --> D["New variable goes out of scope"]
            D --> E["Value dropped"]
            A --> F["Original variable goes out of scope"]
            F --> E
        `}</Diagram>


        ## The Stack and the Heap


        Data with a known size at compile time is stored on the stack. Data with
        an unknown or dynamic size is stored on the heap. In Rust, the ownership
        rules determine when heap data is freed.


        ## Move and Copy


        When you assign a value to another variable, Rust either copies or moves
        the data. For types that implement the `Copy` trait, like integers and
        booleans, the assignment copies the value. For types like `String`, the
        assignment moves the value, transferring ownership.


        The following example shows a value being moved into a function. After
        the call, the original variable is no longer usable.


        <ProgrammingExampleEditor lang="rust"
        name="uadl_01M1A1YCQ4T6CSEHDQB42VYD2H_01">
          <ExampleFile path="example.rs" role="subject" title="Ownership in action" description="Moving a String transfers ownership.">{`fn main() {
            let greeting = String::from("hello");
            take_ownership(greeting);
        }


        fn take_ownership(s: String) {
            println!("{}", s);
        }`}</ExampleFile>

        </ProgrammingExampleEditor>


        The variable `greeting` is moved into `take_ownership`, so referencing
        `greeting` after the call would trigger a compile error.


        ## Borrowing with References


        Borrowing lets you use a value without taking ownership. A reference is
        written with `&`. References do not change ownership; they only borrow
        the value temporarily.


        The diagram below shows the relationship between an owner and a
        reference:


        <Diagram type='mermaid' name="uadl_01M1A1YCQ4T6CSEHDQB42VYD2H_02">{`

        flowchart LR
            A["Owner: s"] --> B["Reference: &s"]
            B --> C["Function uses reference"]
            C --> D["Reference goes out of scope"]
            D --> A["s still valid"]
        `}</Diagram>


        A reference can be passed to a function without transferring ownership:


        ```rust

        let s = String::from("hello");

        let len = calculate_length(&s);

        println!("{}, {}", s, len);


        fn calculate_length(s: &String) -> usize {
            s.len()
        }

        ```


        Here, `s` remains valid after the call because the function only borrows
        it.


        ## Mutable References


        To modify a value through a reference, use a mutable reference: `&mut`.
        While a mutable reference exists, no other references to that value are
        allowed.


        ```rust

        let mut s = String::from("hello");

        let r = &mut s;

        r.push_str(", world");

        println!("{}", r);

        ```


        The rules for borrowing are:


        - At any given time, you can have either one mutable reference or any
        number of immutable references.

        - References must always be valid.


        ## Common Error Patterns


        A common mistake is trying to use a value after it has been moved. The
        compiler rejects such code with an error like "value borrowed after
        move." Another mistake is holding a reference to a value that goes out
        of scope; the borrow checker catches this as a dangling reference.


        ## Practice


        Now it's time to put these ideas into practice. Write a small program
        that reads a line of text and counts the words in it.


        <ProgrammingExercise lang="rust"
        name="uadl_01M1A1YCQ4T6CSEHDQB42VYD2H_03">
          <Statement>{`Read a line from standard input that contains words separated by single spaces. Print an integer representing the number of words in the line.`}</Statement>
          <TestCases>
            <TestCase name="test1">
              <Input>hello world</Input>
              <Output>2</Output>
            </TestCase>
            <TestCase name="test2">
              <Input>rust ownership</Input>
              <Output>2</Output>
            </TestCase>
            <TestCase name="test3">
              <Input>one two three four</Input>
              <Output>4</Output>
            </TestCase>
          </TestCases>
        </ProgrammingExercise>
---

- The rendering for level selection of Programming exercises in Gym does not have Breadcrumbs 

---

- The exercises generation takes too long.

---

- Rust exercises don't work:

VM1125:2 Uncaught TypeError: Cannot read properties of undefined (reading 'startTime')
    at et.reportAllChanges (<anonymous>:2:19429)
    at <anonymous>:2:13070
    at <anonymous>:2:331
    at d (<anonymous>:2:6141)
    at <anonymous>:2:6326
    at <anonymous>:2:2895
    at n.timeout (<anonymous>:2:5652)

---

- Challenge don't work:

(node:1) Warning: Setting the NODE_TLS_REJECT_UNAUTHORIZED environment variable to '0' makes TLS connections and HTTPS requests insecure by disabling certificate verification.
(Use `node --trace-warnings ...` to show where the warning was created)
⨯ Error: execution exceeded 20s timeout
    at c.RunnerdService.runCodeWithTests (.next/server/chunks/ssr/_04iam3s._.js:1:18003)
    at async j (.next/server/chunks/ssr/_04iam3s._.js:1:13659)
    at async h.validateAnswer (.next/server/chunks/ssr/_04iam3s._.js:1:13231)
    at async o.test (.next/server/chunks/ssr/[root-of-the-server]__0mupofo._.js:55:15140)
    at async j.test (.next/server/chunks/ssr/_1amts5o._.js:1:2112)
    at async test (.next/server/chunks/ssr/_1amts5o._.js:1:6627) {
  digest: '496417080'
}
⨯ Error: progLang required
    at c.ChallengeProducer.forgeAndStore (.next/server/chunks/ssr/packages_1063xjy._.js:4:771)
    at async h (.next/server/chunks/ssr/[root-of-the-server]__0-0qmgm._.js:2:10149)
    at async l (.next/server/chunks/ssr/0_us_next_dist_008g4s9._.js:1:10953)
    at async o (.next/server/chunks/ssr/0_us_next_dist_008g4s9._.js:2:4523) {
  digest: '3838794340'
}

--

- Spaces: don't work

[auth][warn][debug-enabled] Read more: https://warnings.authjs.dev
Error: &database not provided, references: rust
    at <unknown> (.next/server/chunks/_0n5hq4y._.js:144:43363)
    at Array.forEach (<anonymous>)
    at e.doRender (.next/server/chunks/_0n5hq4y._.js:144:43336)
    at e.render (.next/server/chunks/_0n5hq4y._.js:144:42900)
    at e.renderMatchingTemplate (.next/server/chunks/_0n5hq4y._.js:144:42741)
    at T.createEnvironmentCRD (.next/server/chunks/_1bhrnv4._.js:9:22020)
    at T.doCreate (.next/server/chunks/_1bhrnv4._.js:9:21201)
    at T.createEnviromentIfNotExists (.next/server/chunks/_1bhrnv4._.js:9:20815)
    at async T.provision (.next/server/chunks/_1bhrnv4._.js:9:19503)
Error: &database not provided, references: rust
    at <unknown> (.next/server/chunks/_0n5hq4y._.js:144:43363)
    at Array.forEach (<anonymous>)
    at e.doRender (.next/server/chunks/_0n5hq4y._.js:144:43336)
    at e.render (.next/server/chunks/_0n5hq4y._.js:144:42900)
    at e.renderMatchingTemplate (.next/server/chunks/_0n5hq4y._.js:144:42741)
    at T.createEnvironmentCRD (.next/server/chunks/_1bhrnv4._.js:9:22020)
    at T.doCreate (.next/server/chunks/_1bhrnv4._.js:9:21201)
    at T.createEnviromentIfNotExists (.next/server/chunks/_1bhrnv4._.js:9:20815)
    at async T.provision (.next/server/chunks/_1bhrnv4._.js:9:19503)