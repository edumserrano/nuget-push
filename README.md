# NuGet push GitHub action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-NuGet%20push-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAOCAYAAAAfSC3RAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAAM6wAADOsB5dZE0gAAABl0RVh0U29mdHdhcmUAd3d3Lmlua3NjYXBlLm9yZ5vuPBoAAAERSURBVCiRhZG/SsMxFEZPfsVJ61jbxaF0cRQRcRJ9hlYn30IHN/+9iquDCOIsblIrOjqKgy5aKoJQj4O3EEtbPwhJbr6Te28CmdSKeqzeqr0YbfVIrTBKakvtOl5dtTkK+v4HfA9PEyBFCY9AGVgCBLaBp1jPAyfAJ/AAdIEG0dNAiyP7+K1qIfMdonZic6+WJoBJvQlvuwDqcXadUuqPA1NKAlexbRTAIMvMOCjTbMwl1LtI/6KWJ5Q6rT6Ht1MA58AX8Apcqqt5r2qhrgAXQC3CZ6i1+KMd9TRu3MvA3aH/fFPnBodb6oe6HM8+lYHrGdRXW8M9bMZtPXUji69lmf5Cmamq7quNLFZXD9Rq7v0Bpc1o/tp0fisAAAAASUVORK5CYII=)](https://github.com/marketplace/actions/nuget-push)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![GitHub Sponsors](https://img.shields.io/github/sponsors/edumserrano)](https://github.com/sponsors/edumserrano)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Eduardo%20Serrano-blue.svg)](https://www.linkedin.com/in/eduardomserrano/)

- [Description](#description)
- [Why should you use this action ?](#why-should-you-use-this-action-)
- [Usage when pushing a single package and corresponding symbols](#usage-when-pushing-a-single-package-and-corresponding-symbols)
- [Usage when pushing multiple packages and corresponding symbols](#usage-when-pushing-multiple-packages-and-corresponding-symbols)
- [Usage when pushing a single NuGet package and symbols package but you don't want to specify the filenames for the packages](#usage-when-pushing-a-single-nuget-package-and-symbols-package-but-you-dont-want-to-specify-the-filenames-for-the-packages)
- [Action inputs](#action-inputs)
- [Action outputs](#action-outputs)
- [Example JSON output from `push-result`](#example-json-output-from-push-result)
  - [Some notes regarding the JSON object from the `push-result` output](#some-notes-regarding-the-json-object-from-the-push-result-output)
- [Action exit status codes](#action-exit-status-codes)
- [Action logs](#action-logs)
- [Dev notes](#dev-notes)

## Description

A composite [GitHub action](https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions) that can be used to push NuGet packages and symbols to [nuget.org](https://www.nuget.org/).

## Why should you use this action ?

I started by pushing my NuGet package and symbols using a command such as:

```
dotnet nuget push ./*.nupkg --api-key <api-key> --source https://api.nuget.org/v3/index.json --skip-duplicate
```

The above command has a flag set to skip publishing the NuGet (.nupgk) if the version has already been publish. This allowed the workflow to run without failing even if it didn't wproduce a new version of the package.

However there was an issue with this approach in that even if the NuGet package already existed the `--skip-duplicate` flag only makes it so that the `nuget push` command doesn't fail due to the returned `409` from the server **but it still tries to push the symbols package after**.

The above doesn't fail but it makes *nuget.org* send emails to the owner of the package with the following:

```
Symbols package publishing failed. The associated symbols package could not be published due to the following reason(s):
The uploaded symbols package contains pdb(s) for a corresponding dll(s) not found in the NuGet package.
Once you've fixed the issue with your symbols package, you can re-upload it.

Please note: The last successfully published symbols package is still available for debugging and download.
```

The above error message is also displayed on the NuGet's package page though it's only visible to the owner of the package.

For more information about this see:

- [dotnet nuget push with --skip-duplicate pushes .snupkg constantly and causes validation to fail.](https://github.com/NuGet/Home/issues/10475)
- [When nupkg exists on push --skip-duplicate, don't automatically push snupkg](https://github.com/NuGet/Home/issues/9647)
- [[Symbols] Support removing snupkg validation error messages](https://github.com/NuGet/NuGetGallery/issues/8036)

This action was created to avoid this from happening. This action pushes the NuGet package and only if it succeeds attempts to do a following push of the corresponding symbols package.

:warning: The main reason for using this action has now been fixed in the nuget tool. See the GitHub issues linked above. **Even so, this action still has value:**

- It provides better feedback on the result of pushing NuGet packages, specially when you push multiple packages.
- It allows you to control if the NuGet push operation should result in a failure if the package already exists or not. See the `fail-if-exists` [action input parameter](#action-inputs).


## Usage when pushing a single package and corresponding symbols

If you only want to push a single NuGet package and corresponding symbols you can do as shown in the example below. You must at least specify the `nuget-package` input parameter, if you don't have a symbols package you don't need to use the `symbols-package` input parameter.

```yml
- name: Publish NuGet and symbols
  id: nuget-push
  uses: edumserrano/nuget-push@v1
  with:
    api-key: '${{ secrets.NUGET_PUSH_API_KEY }}' # this example is using GitHub secrets to pass the API key
    nuget-package: 'my-awesome-package.nupkg'
    symbols-package: 'my-awesome-package.snupkg'
# The next step is using powershell to parse the action's output but you can use whatever you prefer.
# Note that in order to read the step outputs the action step must have an id.
- name: Log output
  if: steps.nuget-push.conclusion != 'skipped' && always() # run regardless if the previous step failed or not, just as long as it wasn't skipped
  shell: pwsh
  run: |
    $pushResult = '${{ steps.nuget-push.outputs.push-result }}' | ConvertFrom-Json
    $pushResultAsJsonIndented = ConvertTo-Json $pushResult
    Write-Output $pushResultAsJsonIndented  # outputs the result of the nuget push operation as an indented JSON string
    Write-Output '${{ steps.nuget-push.outputs.status }}' # outputs the overall status of the nuget push action

    # since we only pushed one package/symbols there's no need to iterate the packages list
    # there will only be one element in the array
    $package = $pushResult.packages[0]
    Write-Output "$($package.status)"  # outputs the status of the nuget push operation
    Write-Output "$($package.package)" # outputs the NuGet package name that was pushed
    Write-Output "$($package.symbols)" # outputs the symbols package name that was pushed
```

## Usage when pushing multiple packages and corresponding symbols

If you want to push several NuGet packages and symbols in one go then you can just specify the directory where the packages are as shown in the example below. The action will pair NuGet packages with their symbols based on their filenames. If there are no matching symbols packages then only the NuGet packages are uploaded.

As an example consider that the `my-packages-dir` contained the following files:

- my-awesome-package.nupkg
- my-awesome-package.snupkg
- my-super-package.nupkg
- my-super-package.snupkg

This action will pair `my-awesome-package.nupkg` with `my-awesome-package.snupkg` and `my-super-package.nupkg` with `my-super-package.snupkg`.

```yml
- name: Publish NuGet and symbols
  id: nuget-push
  uses: edumserrano/nuget-push@v1
  with:
    api-key: '${{ secrets.NUGET_PUSH_API_KEY }}' # this example is using GitHub secrets to pass the API key
    working-directory: 'my-packages-dir'
# The next step is using powershell to parse the action's output but you can use whatever you prefer.
# Note that in order to read the step outputs the action step must have an id.
- name: Log output
  if: steps.nuget-push.conclusion != 'skipped' && always() # run regardless if the previous step failed or not, just as long as it wasn't skipped
  shell: pwsh
  run: |
    $pushResult = '${{ steps.nuget-push.outputs.push-result }}' | ConvertFrom-Json
    $pushResultAsJsonIndented = ConvertTo-Json $pushResult
    Write-Output $pushResultAsJsonIndented  # outputs the result of the nuget push operation as an indented JSON string
    Write-Output '${{ steps.nuget-push.outputs.status }}' # outputs the overall status of the nuget push action

    # iterates over all list of packages and outputs the data from the nuget push operation for each
    foreach($package in $pushResult.packages) {
        Write-Output "$($package.status)"  # outputs the status of the nuget push operation
        Write-Output "$($package.package)" # outputs the NuGet package name that was pushed
        Write-Output "$($package.symbols)" # outputs the symbols package name that was pushed
    }
```

## Usage when pushing a single NuGet package and symbols package but you don't want to specify the filenames for the packages

You can use the `working-directory` input parameter to accomplish this. Just make sure that the given directory only contains a single NuGet package and potentially the corresponding symbols package.

As an example consider that the `my-packages-dir` contained the following files:

- my-awesome-package.nupkg
- my-awesome-package.snupkg

Then you could do:

```yml
- name: Publish NuGet and symbols
  id: nuget-push
  uses: edumserrano/nuget-push@v1
  with:
    api-key: '${{ secrets.NUGET_PUSH_API_KEY }}' # this example is using GitHub secrets to pass the API key
    working-directory: 'my-packages-dir'
# The next step is using powershell to parse the action's output but you can use whatever you prefer.
# Note that in order to read the step outputs the action step must have an id.
- name: Log output
  if: steps.nuget-push.conclusion != 'skipped' && always() # run regardless if the previous step failed or not, just as long as it wasn't skipped
  shell: pwsh
  run: |
    $pushResult = '${{ steps.nuget-push.outputs.push-result }}' | ConvertFrom-Json
    $pushResultAsJsonIndented = ConvertTo-Json $pushResult
    Write-Output $pushResultAsJsonIndented  # outputs the result of the nuget push operation as an indented JSON string
    Write-Output '${{ steps.nuget-push.outputs.status }}' # outputs the overall status of the nuget push action

    # since we only pushed one package/symbols there's no need to iterate the packages list
    # there will only be one element in the array
    $package = $pushResult.packages[0]
    Write-Output "$($package.status)"  # outputs the status of the nuget push operation
    Write-Output "$($package.package)" # outputs the NuGet package name that was pushed
    Write-Output "$($package.symbols)" # outputs the symbols package name that was pushed
```

## Action inputs

<!-- the &nbsp; is a trick to expand the width of the table column. You add as many &nbsp; as required to get the width you want. -->
| Name &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; | Description                                                                                                                                                                                                                                         | Required | Default value |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------- |
| `api-key`                                                                                                                                                                   | The API key for the NuGet server. Used when pushing the NuGet packages and symbols.                                                                                                                                                                 | yes      | -             |
| `fail-if-exists`                                                                                                                                                            | Indicates whether this actions should fail if the version of the NuGet package being pushed already exists in the server. If multiple NuGet packages are being pushed it will fail if a single one already exists.                                  | no       | false         |
| `working-directory`                                                                                                                                                         | The directory that will be used to push NuGet packages. It will push all NuGet packages (\*.nupkg) and corresponding symbol packages (\*.snupkg) present in the directory. Cannot be specified in combination with `nuget-package` input parameter. | no       | -             |
| `nuget-package`                                                                                                                                                             | The filepath for the NuGet package to be pushed. Cannot be specified in combination with `working-directory` input parameter.                                                                                                                       | no       | -             |
| `symbols-package`                                                                                                                                                           | The filepath for the symbols package to be pushed. To be used in combination with `nuget-package` input parameter or not at all.                                                                                                                    | no       | -             |

## Action outputs

<!-- the &nbsp; is a trick to expand the width of the table column. You add as many &nbsp; as required to get the width you want. -->
| Name &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; | Description                                                                                                      |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `status`                                                            | The overall status of pushing the NuGet packages and corresponding symbols. Possible values are `ok` or `error`. |
| `push-result`                                                       | The result of pushing the NuGet packages and corresponding symbols as a JSON string.                             |

## Example JSON output from `push-result`

Let's assume you used the action to upload the following packages from a some directory:

- my-awesome-package.nupkg
- my-awesome-package.snupkg
- my-super-package.nupkg
- my-super-package.snupkg

Let's also assume that the `my-awesome-package` package was pushed successfully and that the `my-super-package` package wasn't pushed because the same version of that package already existed. In this example the output would be:

```json
{
  "packages": [
    {
      "package": "my-awesome-package.nupkg",
      "status": "ok",
      "symbols": "my-awesome-package.snupkg"
    }
    {
      "package": "my-super-package.nupkg",
      "status": "nuget-already-exists",
      "symbols": "my-super-package.snupkg"
    }
  ],
  "status": "ok"
}
```

### Some notes regarding the JSON object from the `push-result` output

- The possible values for the `status` property at the root of the JSON object are either `ok` or `error`. In fact the value of this property will always match the value of the `status` action output. The `status` action output parameter was added explicitly as an output for convenience.
- The array represented by the `packages` property will contain information about all the NuGet packages and corresponding symbols that were pushed.
- Each object in the `packages`  property array will have always three properties:
  - `package`: the NuGet package that was pushed.
  - `symbol`: the symbols package that was pushed, if any. It will be an empty string if no symbols package was pushed.
  - `status`: the result of pushing the NuGet package and corresponding symbols. The possible values for this property are:
    - `ok`: success pushing the NuGet package and symbols.
    - `nuget-already-exists`: the version of the NuGet package already exists in the NuGet server. When this happens the corresponding symbols package, if exists, will not even be attempted to be pushed.
    - `nuget-push-failed`: an error occurred when pushing the NuGet package.
    - `symbols-push-failed`: an error occurred when pushing the symbols package.

## Action exit status codes

This action will set the exit status code based on its `status` output parameter. Meaning that the action will complete successfully when the `status` output parameter is `ok` and it will fail when it's `error`.

If this behavior is not desirable and you always want to check the `push-result` output then you can set the `continue-on-error` property to make sure the action never fails and then analyze the output from the action in a following step as shown below.

```yml
- name: Publish NuGet and symbols
  id: nuget-push
  uses: edumserrano/nuget-push@v1
  continue-on-error: true
  with:
    api-key: '${{ secrets.NUGET_PUSH_API_KEY }}'
    nuget-package: 'my-awesome-package.nupkg'
    symbols-package: 'my-awesome-package.snupkg'
# The next step is using powershell to parse the action's output but you can use whatever you prefer.
# Note that in order to read the step outputs the action step must have an id.
- name: Do something based on the output of the nuget push action
  shell: pwsh
  run: |
    $pushResult = '${{ steps.nuget-push.outputs.push-result }}' | ConvertFrom-Json
    $pushResultAsJsonIndented = ConvertTo-Json $pushResult
    Write-Output $pushResultAsJsonIndented  # outputs the result of the nuget push operation as an indented JSON string
    Write-Output '${{ steps.nuget-push.outputs.status }}' # outputs the overall status of the nuget push action
    # do soemthing based on the output from the nuget-push step
    ...
```

## Action logs

The action will output detailed information regarding what packages it's going to push and what was the result of each push operation. Check the action logs for this information.

## Dev notes

For notes aimed at developers working on this repo or just trying to understand it go [here](/docs/dev-notes/README.md).
