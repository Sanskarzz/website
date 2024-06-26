---
title: "Require Replicas Allow Disruption"
category: Other
version: 
subject: PodDisruptionBudget, Deployment, StatefulSet
policyType: "validate"
description: >
    Existing PodDisruptionBudgets can apply to all future matching Pod controllers. If the minAvailable field is defined for such matching PDBs and the replica count of a new Deployment or StatefulSet is lower than that, then availability could be negatively impacted. This policy specifies that Deployment/StatefulSet replicas exceed the minAvailable value of all matching PodDisruptionBudgets which specify minAvailable as a number and not percentage.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/require-replicas-allow-disruption/require-replicas-allow-disruption.yaml" target="-blank">/other/require-replicas-allow-disruption/require-replicas-allow-disruption.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-replicas-allow-disruption
  annotations:
    policies.kyverno.io/title: Require Replicas Allow Disruption
    policies.kyverno.io/category: Other
    kyverno.io/kyverno-version: 1.11.4
    kyverno.io/kubernetes-version: "1.27"
    policies.kyverno.io/subject: PodDisruptionBudget, Deployment, StatefulSet
    policies.kyverno.io/description: >-
      Existing PodDisruptionBudgets can apply to all future matching Pod controllers.
      If the minAvailable field is defined for such matching PDBs and the replica count of a new
      Deployment or StatefulSet is lower than that, then availability could be negatively impacted.
      This policy specifies that Deployment/StatefulSet replicas exceed the minAvailable value of all
      matching PodDisruptionBudgets which specify minAvailable as a number and not percentage.
spec:
  validationFailureAction: Audit
  background: false
  rules:
  - name: replicas-check
    match:
      any:
      - resources:
          kinds:
          - Deployment
          - StatefulSet
          operations:
          - CREATE
          - UPDATE
    context:
    - name: matchingpdbs
      apiCall:
        jmesPath: items[?label_match(spec.selector.matchLabels, `{{request.object.spec.template.metadata.labels}}`)]
        urlPath: /apis/policy/v1/namespaces/{{request.namespace}}/poddisruptionbudgets
    preconditions:
      all:
      - key: "{{ request.object.spec.replicas }}"
        operator: GreaterThan
        value: 0
      - key: "{{ length(matchingpdbs) }}"
        operator: GreaterThan
        value: 0
    validate:
      message: >-
        Replica count ({{ request.object.spec.replicas }}) cannot be less than or equal to the minAvailable of any
        matching PodDisruptionBudget. There are {{ length(matchingpdbs) }} PodDisruptionBudgets which match this labelSelector,
        not all of which may define a minAvailable value as a number.
      foreach:
      - list: matchingpdbs
        preconditions:
          all:
          - key: '{{ regex_match(''^[0-9]+$'', ''{{ element.spec.minAvailable || '''' }}'') }}'
            operator: Equals
            value: true
        deny:
          conditions:
            all:
            - key: "{{ request.object.spec.replicas }}"
              operator: LessThanOrEquals
              value: "{{ element.spec.minAvailable }}"

```
