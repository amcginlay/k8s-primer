---
title: "Linux Namespaces - Appendix"
chapter: false
weight: 900
draft: false
---

## More on PID Namespace Hierarchy

In the Linux Namespaces section, you created separate PID namespaces for isolated trees of processes. This was the graphical depiction we showed then.


{{< mermaid >}}
graph TB
    subgraph pid-namespace-zero
         proc1((PID 5830<br>bash<br>shell1))
         proc2((PID 5831<br>bash<br>shell2))
    end
    subgraph pid-namespace-one
         proc3((PID 1<br>bash))
         proc4((PID 15<br>sleep 1001))
         proc5((PID 16<br>ps -ef))
    end
    subgraph pid-namespace-two
         proc6((PID 1<br>bash))
         proc7((PID 15<br>sleep 1002))
         proc8((PID 16<br>ps -ef))
    end
proc1 -->|unshare --pid| proc3
proc2 -->|unshare --pid| proc6
proc3 --> proc4 & proc5
proc6 --> proc7 & proc8

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
class proc1,proc2 blue;
class proc3,proc4,proc5 green;
class proc6,proc7,proc8 orange;
{{< /mermaid >}}

This is not the whole truth. If you're curious for further details, read on.

These PID namespaces are isolated from each other, but not from their parents or further ancestor PID namespaces.
Each process has a PID within its ancestor namespaces.

Michael Kerrisk describes this well and depicts it in a diagram in [Containers unplugged: Linux namespaces](https://www.youtube.com/watch?v=0kJPa-1FuoI). Here is our own rendering of such a diagram.

{{< mermaid >}}
graph TB
    subgraph pid-namespace-zero[Parent PID Namespace B]
         proc0((PID 1))
         proc1((PID 5830<br>bash<br>shell1))
         proc3-0((PID 5832<br>bash))
         proc4-0((PID 5847<br>sleep 1001))
         proc5-0((PID 5862<br>ps -ef))
         proc2((PID 5831<br>bash<br>shell2))
         proc6-0((PID 5833<br>bash))
         proc7-0((PID 5861<br>sleep 1002))
         proc8-0((PID 5863<br>ps -ef))
    end
    subgraph pid-namespace-one[Child PID Namespace G]
         proc3((PID 1<br>bash))
         proc4((PID 15<br>sleep 1001))
         proc5((PID 16<br>ps -ef))
    end
    subgraph pid-namespace-two[Child PID Namespace R]
         proc6((PID 1<br>bash))
         proc7((PID 15<br>sleep 1002))
         proc8((PID 16<br>ps -ef))
    end
proc0 --> proc1
proc0 --> proc2
proc3 -.- proc3-0
proc4 -.- proc4-0
proc5 -.- proc5-0
proc6 -.- proc6-0
proc7 -.- proc7-0
proc8 -.- proc8-0
proc1 -->|unshare --pid| proc3
proc2 -->|unshare --pid| proc6
proc3 --> proc4 & proc5
proc6 --> proc7 & proc8

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blueShadow fill:#0cf,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5;
class proc0,proc1,proc2 blue;
class proc3,proc4,proc5 green;
class proc6,proc7,proc8 orange;
class proc3-0,proc4-0,proc5-0,proc6-0,proc7-0,proc8-0 blueShadow;
{{< /mermaid >}}

The processes in the child namespaces Green (G) and Red (R) not only have process identifiers (PIDs) which are unique only within their own namespaces, but *also* have PID in their parent namespace--the Blue (B) namespace--which are unique within *that* (parent) namespace. If there are grandchild namespaces, those processes can also be viewed from their parent *and* grandparent PID namespaces, with unique PIDs in each.

