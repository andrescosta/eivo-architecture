* EvBooks:
1- Define UX with styles
2- Define the component for Exercise <COMPLETED>
3- Generation:
  3.1- Creates the prompt supporting one file per lesson <COMPLETED>
  3.2- Update crafter to generate MDX files and prompts <COMPLETED>
4- Integration in Lingv
  4.1- Learning UI <WORKING>
  4.2- Support one file per lesson. <COMPLETED>
  4.3- Support transparent scrolling.
  4.4- In exercises, look for a tag called exercise.
5- Security
  5.1- Content sanitizer:
    5.1.1. rehype-sanitize (Recommended for MDX)
    This is specifically designed for MDX/HTML content and integrates directly with the MDX processing pipeline DhiWiseGitHub. It's part of the unified.js ecosystem that MDX uses.
    5.1.2. DOMPurify (Most Popular)
    DOMPurify is a DOM-only, super-fast, uber-tolerant XSS sanitizer for HTML, MathML and SVG with over 10 million weekly downloads GitHubnpm Trends.
    5.1.3. sanitize-html (Lightweight Alternative)
    This has over 3 million weekly downloads and offers good performance dompurify vs rehype-sanitize vs sanitize-html | npm trends for basic HTML sanitization.

* Learning class:
 init the session if not exist, or hidratate it from backend.
   as part the init, init the infinite queue for the lectures
 list the session already started but not completed.
 list the session completed with statistics.
* Learning session:
- are state full
- track: 
     completitions (where I left)
     evaluations
     problems
- give the possibility to highlights content
- track any statitiscs (time consumed, etc.)
- the session is associated to an aggregate, that potentially could contains more aggregate
- the content is a lecture
- it must provide the capability of getting the content(lecture) by aggregate id
- it must provide the possibility to cancel, reset it, mark it as completed
- it must provide the possibility to validate the result of exercises.
- it must support evaluations that are part of a lecture, track its result.
- it must call the renders to render the lecture

* Next step
- review/recover old ideas

* What I need:
** Social API:
- Start a class and associate the student to it
- Register activities (where I left, evaluations, etc.)
- Register a student to a class
- Get class/students in class
- Get a lecture by parents id
** Lingv:
- Infinite queue for lectures
 - Lecture rendering(MDX) Foundry ?

** Specs:
 - Desfine evaluation, content and exercises in coexistence as part of a lesson
 
* Req:
- All the social API must support namespace

* Idea:
 - For Lingv, there will be one class per aggregate. 

 * UI