# Dev notes

## Composite GitHub action

This repo provides a composite action. See the [docs](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) for more information on how to build a composite action.

The entire composite action consists of inline powershell scripts. The powershell script could be made easier to work with and modularized (use functions for instance) if the powershell wasn't used inline and instead was moved to a file. Then the action could also be simplified to become a single step that invokes the powershell script with the input parameters from the action.

## GitHub marketplace

This action is published to the [GitHub marketplace](https://github.com/marketplace/actions/nuget-push). See here for more information on [how to publish or remove an action from the marketplace](https://docs.github.com/en/actions/creating-actions/publishing-actions-in-github-marketplace).

**Currently there is no workflow setup to publish this action to the marketplace. The publishing act is a manual process following the instructions above.**
