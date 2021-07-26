---
title:  "AWS CDK - Computing diff between infrastructure templates [TypeScript]"
layout: default
language: TypeScript
project: AWS CDK
project-key: aws-cdk
tags: aws,cloud,diff,infrastructure-as-code
parent: Articles
---

# Computing diff between infrastructure templates in AWS CDK

* **Status**: DRAFT
* **Project name**: AWS CDK
* **Example name**: Computing diff between infrastructure templates
* **Project home page**: https://github.com/aws/aws-cdk
* **Programming language(s)**: TypeScript
* **Frameworks, libraries used:** N/A
* **Tags:** aws,cloud,diff,infrastructure-as-code

## Context

*Give a high-level description of the project. Explain the part(s) of it that include the example in more detail.*

CDK (Cloud Development Kit) is an [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code) software tool created by [AWS](https://aws.amazon.com/).

CDK is used to synthesize and deploy [CloudFormation](https://aws.amazon.com/cloudformation/) infrastructure templates. 

Infrastructures are made up of resources (virtual machines, database tables, load balancers, etc.) Resources can depend on other resources.

## Problem

Synthesized infrastructure templates need to be compared to the existing state of the infrastructure, to see what resources will be created, updated or deleted if the template is deployed. The diff algorithm needs to be aware of the template semantics.

## Overview

There are different diff handlers for 9 top-level keys (`AWSTemplateFormatVersion`, `Description`, `Metadata`, `Parameters`, `Mappings`, `Conditions`, `Transform`, `Resources`, `Outputs`).

It calculates what was added, removed or updated. For each changed resource it decides the impact: if it will be updated, destroyed, orphaned (excluded from the template but not actually deleted).

Changes to one resource can trigger changes to resourced dependant on it. These changes are propagated until convergence.

The diff can be visualized.

## Implementation details

[Main method](https://github.com/aws/aws-cdk/blob/d88b45eb21bcd051146477e3c97de7dd7b8634d3/packages/%40aws-cdk/cloudformation-diff/lib/diff-template.ts#L31-L78):

```typescript
/**
 * Compare two CloudFormation templates and return semantic differences between them.
 *
 * @param currentTemplate the current state of the stack.
 * @param newTemplate     the target state of the stack.
 *
 * @returns a +types.TemplateDiff+ object that represents the changes that will happen if
 *      a stack which current state is described by +currentTemplate+ is updated with
 *      the template +newTemplate+.
 */
export function diffTemplate(currentTemplate: { [key: string]: any }, newTemplate: { [key: string]: any }): types.TemplateDiff {
  // Base diff
  const theDiff = calculateTemplateDiff(currentTemplate, newTemplate);

  // We're going to modify this in-place
  const newTemplateCopy = deepCopy(newTemplate);

  let didPropagateReferenceChanges;
  let diffWithReplacements;
  do {
    diffWithReplacements = calculateTemplateDiff(currentTemplate, newTemplateCopy);

    // Propagate replacements for replaced resources
    didPropagateReferenceChanges = false;
    if (diffWithReplacements.resources) {
      diffWithReplacements.resources.forEachDifference((logicalId, change) => {
        if (change.changeImpact === types.ResourceImpact.WILL_REPLACE) {
          if (propagateReplacedReferences(newTemplateCopy, logicalId)) {
            didPropagateReferenceChanges = true;
          }
        }
      });
    }
  } while (didPropagateReferenceChanges);

  // Copy "replaced" states from `diffWithReplacements` to `theDiff`.
  diffWithReplacements.resources
    .filter(r => isReplacement(r!.changeImpact))
    .forEachDifference((logicalId, downstreamReplacement) => {
      const resource = theDiff.resources.get(logicalId);

      if (resource.changeImpact !== downstreamReplacement.changeImpact) {
        propagatePropertyReplacement(downstreamReplacement, resource);
      }
    });

  return theDiff;
}
```

[Diffing templates (without propagation)](https://github.com/aws/aws-cdk/blob/d88b45eb21bcd051146477e3c97de7dd7b8634d3/packages/%40aws-cdk/cloudformation-diff/lib/diff-template.ts#L96-L111):
```typescript
function calculateTemplateDiff(currentTemplate: { [key: string]: any }, newTemplate: { [key: string]: any }): types.TemplateDiff {
  const differences: types.ITemplateDiff = {};
  const unknown: { [key: string]: types.Difference<any> } = {};
  for (const key of unionOf(Object.keys(currentTemplate), Object.keys(newTemplate)).sort()) {
    const oldValue = currentTemplate[key];
    const newValue = newTemplate[key];
    if (deepEqual(oldValue, newValue)) { continue; }
    const handler: DiffHandler = DIFF_HANDLERS[key]
                  || ((_diff, oldV, newV) => unknown[key] = impl.diffUnknown(oldV, newV));
    handler(differences, oldValue, newValue);

  }
  if (Object.keys(unknown).length > 0) { differences.unknown = new types.DifferenceCollection(unknown); }

  return new types.TemplateDiff(differences);
}
```

[Diffing two resources](https://github.com/aws/aws-cdk/blob/d88b45eb21bcd051146477e3c97de7dd7b8634d3/packages/%40aws-cdk/cloudformation-diff/lib/diff/index.ts#L29-L77):

```typescript
export function diffResource(oldValue?: types.Resource, newValue?: types.Resource): types.ResourceDifference {
  const resourceType = {
    oldType: oldValue && oldValue.Type,
    newType: newValue && newValue.Type,
  };
  let propertyDiffs: { [key: string]: types.PropertyDifference<any> } = {};
  let otherDiffs: { [key: string]: types.Difference<any> } = {};

  if (resourceType.oldType !== undefined && resourceType.oldType === resourceType.newType) {
    // Only makes sense to inspect deeper if the types stayed the same
    const typeSpec = cfnspec.filteredSpecification(resourceType.oldType);
    const impl = typeSpec.ResourceTypes[resourceType.oldType];
    propertyDiffs = diffKeyedEntities(oldValue!.Properties,
      newValue!.Properties,
      (oldVal, newVal, key) => _diffProperty(oldVal, newVal, key, impl));

    otherDiffs = diffKeyedEntities(oldValue, newValue, _diffOther);
    delete otherDiffs.Properties;
  }

  return new types.ResourceDifference(oldValue, newValue, {
    resourceType, propertyDiffs, otherDiffs,
  });

  function _diffProperty(oldV: any, newV: any, key: string, resourceSpec?: cfnspec.schema.ResourceType) {
    let changeImpact = types.ResourceImpact.NO_CHANGE;

    const spec = resourceSpec && resourceSpec.Properties && resourceSpec.Properties[key];
    if (spec && !deepEqual(oldV, newV)) {
      switch (spec.UpdateType) {
        case cfnspec.schema.UpdateType.Immutable:
          changeImpact = types.ResourceImpact.WILL_REPLACE;
          break;
        case cfnspec.schema.UpdateType.Conditional:
          changeImpact = types.ResourceImpact.MAY_REPLACE;
          break;
        default:
          // In those cases, whatever is the current value is what we should keep
          changeImpact = types.ResourceImpact.WILL_UPDATE;
      }
    }

    return new types.PropertyDifference(oldV, newV, { changeImpact });
  }

  function _diffOther(oldV: any, newV: any) {
    return new types.Difference(oldV, newV);
  }
}
```

[Rendering in a human-readable form](https://github.com/aws/aws-cdk/blob/d88b45eb21bcd051146477e3c97de7dd7b8634d3/packages/%40aws-cdk/cloudformation-diff/lib/format.ts)

## Testing


[The test suite](https://github.com/aws/aws-cdk/blob/d88b45eb21bcd051146477e3c97de7dd7b8634d3/packages/%40aws-cdk/cloudformation-diff/test/diff-template.test.ts) is quite comprehensive.

Basic test for when a resource is added:
```typescript
test('when a resource is created', () => {
  const currentTemplate = { Resources: {} };

  const newTemplate = { Resources: { BucketResource: { Type: 'AWS::S3::Bucket' } } };

  const differences = diffTemplate(currentTemplate, newTemplate);
  expect(differences.differenceCount).toBe(1);
  expect(differences.resources.differenceCount).toBe(1);
  const difference = differences.resources.changes.BucketResource;
  expect(difference).not.toBeUndefined();
  expect(difference?.isAddition).toBeTruthy();
  expect(difference?.newResourceType).toEqual('AWS::S3::Bucket');
  expect(difference?.changeImpact).toBe(ResourceImpact.WILL_CREATE);
});
```

Test cascading changes:
```typescript
test('resource replacement is tracked through references', () => {
  // If a resource is replaced, then that change shows that references are
  // going to change. This may lead to replacement of downstream resources
  // if the reference is used in an immutable property, and so on.

  // GIVEN
  const currentTemplate = {
    Resources: {
      Bucket: {
        Type: 'AWS::S3::Bucket',
        Properties: { BucketName: 'Name1' }, // Immutable prop
      },
      Queue: {
        Type: 'AWS::SQS::Queue',
        Properties: { QueueName: { Ref: 'Bucket' } }, // Immutable prop
      },
      Topic: {
        Type: 'AWS::SNS::Topic',
        Properties: { TopicName: { Ref: 'Queue' } }, // Immutable prop
      },
    },
  };

  // WHEN
  const newTemplate = {
    Resources: {
      Bucket: {
        Type: 'AWS::S3::Bucket',
        Properties: { BucketName: 'Name2' },
      },
      Queue: {
        Type: 'AWS::SQS::Queue',
        Properties: { QueueName: { Ref: 'Bucket' } },
      },
      Topic: {
        Type: 'AWS::SNS::Topic',
        Properties: { TopicName: { Ref: 'Queue' } },
      },
    },
  };
  const differences = diffTemplate(currentTemplate, newTemplate);

  // THEN
  expect(differences.resources.differenceCount).toBe(3);
});
```

Testing that it understands that the order of elements in an array matters in some places and doesn't matter is some other:

```typescript
test('array equivalence is independent of element order in DependsOn expressions', () => {
  // GIVEN
  const currentTemplate = {
    Resources: {
      BucketResource: {
        Type: 'AWS::S3::Bucket',
        DependsOn: ['SomeResource', 'AnotherResource'],
      },
    },
  };

  // WHEN
  const newTemplate = {
    Resources: {
      BucketResource: {
        Type: 'AWS::S3::Bucket',
        DependsOn: ['AnotherResource', 'SomeResource'],
      },
    },
  };

  let differences = diffTemplate(currentTemplate, newTemplate);
  expect(differences.resources.differenceCount).toBe(0);

  differences = diffTemplate(newTemplate, currentTemplate);
  expect(differences.resources.differenceCount).toBe(0);
});

test('arrays of different length are considered unequal in DependsOn expressions', () => {
  // GIVEN
  const currentTemplate = {
    Resources: {
      BucketResource: {
        Type: 'AWS::S3::Bucket',
        DependsOn: ['SomeResource', 'AnotherResource', 'LastResource'],
      },
    },
  };

  // WHEN
  const newTemplate = {
    Resources: {
      BucketResource: {
        Type: 'AWS::S3::Bucket',
        DependsOn: ['AnotherResource', 'SomeResource'],
      },
    },
  };

  let differences = diffTemplate(currentTemplate, newTemplate);
  expect(differences.resources.differenceCount).toBe(1);

  differences = diffTemplate(newTemplate, currentTemplate);
  expect(differences.resources.differenceCount).toBe(1);
});
```

## Observations

*Add more observations about the code.*

TODO

## Related

[Terraform](https://github.com/hashicorp/terraform/), a competing product, [implements a similar algorithm](https://github.com/hashicorp/terraform/blob/d35bc0531255b496beb5d932f185cbcdb2d61a99/internal/legacy/terraform/diff.go).

## References

* [Github Repo](https://github.com/aws/aws-cdk)
* [Product website](https://aws.amazon.com/cdk/)
* [Documentation](https://docs.aws.amazon.com/cdk/index.html)
* [Sample templates](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/sample-templates-services-us-west-2.html)
