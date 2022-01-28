---
title: "Scratch"
disableToc: true
---

## A place to try out Hugo features ...

{{< step >}}
Get your coffee. Always the first step.
{{< /step >}}

Then something else happens.

{{< step >}}
Download the `k8s-primer`.
{{< /step >}}

{{< step >}}
Then launch Hugo.
{{< /step >}}

Now we have to test a longer step.

{{< step >}}
This is the time for all good steps to go in sequence, be properly indented, and maybe even contain a little markdown so we can indicate a `Pod` kind of thing, or a `spec.replicas` sort of notation. No, that was not the only sentence in the paragraph. It just keeps on going.
{{< /step >}}

{{< step >}}And this is yet another step. Remember to load your images. We should build another one besides `demo:1.0.0` to show them how the `Deployment` handles a change of image. That would be a good exercise. I think a separate exercise after the current deployment one.{{< /step >}}

{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}

## Quizdown

Here is the demo quizdown quiz from https://github.com/bonartm/hugo-quiz/README.md:


{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## The sound of dog

---
shuffle_answers: false
---

What do dogs sound like?

> Some hint

- [ ] yes
- [ ] no
- [ ] `self.sound = "meow"`
- [x] wuff

## Put the [days](https://en.wikipedia.org/wiki/Day) in order!

> Monday is the *first* day of the week.

1. Monday
2. Tuesday
3. Wednesday
4. Friday
5. Saturday  
{{< /quizdown >}}

## First scratch Kubernetes quiz

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## What is Kubernetes?

---
shuffle_answers: false
---

Which of the following best describes Kubernetes?

> What is a name for a musical group with saxophone and violins?

- [ ] container runtime
- [x] container orchestrator
- [ ] container packager
- [ ] container image

## Unit of compute

Which of these is the fundamental unit of compute in Kubernetes?

> What do cetaceans travel in? Edamame? Peas?

- [ ] Container
- [x] Pod
- [ ] ReplicaSet
- [ ] Deployment

## Pod provisioning

Sequence these pieces of software in order of operation from your shell to the creation of containers in a pod.

> Trace the path your pod creation request would go through these Kubernetes components.

1. kubectl
2. API server
3. Scheduler
4. kubelet
5. container runtime (e.g. containerd, dockerd, cri-o)
{{< /quizdown >}}

## More on Mermaid

Removed this in the refactoring of the `Deployment` chapter...
...definitely scratch material

## Scratch

Yes, a scratch section at the end of this lesson. I'll hoist one up to top-level soon.

A test diagram to ensure that removing `./static/mermaid/` and the ancient `./static/mermaid/mermaid.min.js` 
has resolved the issue with that old mermaid version blocking the much more recent one in the hugo and hugo learn theme locations. 

{{< mermaid >}}
graph TD
subgraph some-id[Title]
    A --> B
end

style some-id fill:#F77,stroke:#F00,stroke-width:2px
{{< /mermaid >}}

Hurray! It worked. We can now have separate id and title for subgraph and even style subgraphs with the revised mermaid.

Therefore, we can style subgraphs within the `graph` diagram style.

{{< mermaid >}}
graph TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata[Deployment metadata]
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec[Deployment spec]
    replicas(replicas)
    strategy
    selector
    subgraph template
      subgraph Pod-metadata
        podLabels[labels]
      end
      subgraph Pod-spec
        containers
      end
    end
  end
end

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class Deployment-manifest orange;
class Deployment-metadata blue;
class Deployment-spec green;
class template yellow;
{{< /mermaid >}}

And even do the `flowchart` style which has proper direction flow inherited into its subgraphs.
So we can go vertical instead of horizontal with a depiction of the Deployment manifest.

{{< mermaid >}}
flowchart TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata[Deployment metadata]
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec[Deployment spec]
    subgraph spec-top-level[ ]
      replicas(replicas)
      strategy
      selector
    end
    subgraph template
      subgraph Pod-metadata
        podLabels[labels]
      end
      subgraph Pod-spec
        containers
      end
    end
  end
end

style spec-top-level fill:#F77,stroke:#F00,stroke-width:0px,color:#000

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
%% class Deployment-metadata orange;
%% class Deployment-spec yellow;
{{< /mermaid >}}

