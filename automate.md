# Sample commands

This document shows how you can use [Speech CLI with PowerShell](https://learn.microsoft.com/azure/cognitive-services/Speech-Service/spx-basics?tabs=windowsinstall%2Cpowershell#create-a-resource-configuration) to create a script for uploading datasets, training custom models, and running evaluations.

## Set up a key and a region
```
spx --% config @key --set SpeechSubscriptionKey
spx --% config @region --set SpeechServiceRegion
```

## Create a project for custom speech in speech studio

Use the `spx csr project create` command to create a custom speech project that's targeted for the en-US locale. Save the output of the command (just the project URL) to a text file so you can use it in subsequent steps.

```
spx --% csr project create --name "githubsample" --language "en-US" --description "custom speech project for github sample" --output url "@my.project.txt"
```

Successful execution of this command should produce a JSON output that includes the project details.

**Note:** If a project with the specified name already exists, a new project will still be created. You can get a list of projects by going to [Speech Studio](https://speech.microsoft.com/portal) and selecting the custom speech section. Make sure that the right Azure Speech Service is selected.

## Upload datasets

Use the `spx csr dataset upload` command to upload datasets. For a pronunciation dataset, use `kind = pronunciation`. For a plain text dataset, use `kind = language`. For evaluation data that contains audio and transcription as a source of ground truth, use `kind = acoustic`. See [Training and testing datasets](https://learn.microsoft.com/azure/cognitive-services/speech-service/how-to-custom-speech-test-and-train) for information about the dataset types. Save the output (the dataset URL) in a text file for later use.

### Upload pronunciation file

```
spx --% csr dataset upload --name "pronunciation_data" --description "data for pronunciation model" --data .\pronunciation.txt --kind Pronunciation --project "@my.project.txt" --output url "@my.dataset.pm.txt"
```

Use the `spx csr dataset status` command to check the status of the `upload` command. If the dataset uploaded successfully, the output JSON will include details on the number of lines that were accepted and the number of lines that were rejected.

**Get status**

``` 
spx --% csr dataset status --dataset "@my.dataset.pm.txt" -–wait
```

### Upload plain text (for language model adaptation)

```
spx --% csr dataset upload --name "language_model_data" --description "data for language model adaptation" --data .\language_model.txt --kind Language --project "@my.project.txt" --output url "@my.dataset.lm.txt"
```

**Get status**
```
spx --% csr dataset status --dataset "@my.dataset.lm.txt" --wait
```

### Upload evaluation data

To run an evaluation (word error rate calculation) on the custom model, you need evaluation or test data. This data will include the audio file and ground truth transcription zipped into one file.

```
spx --% csr dataset upload --name "test_data_1" --description "Test data_1 for evaluation of custom model" --data .\test_data_1.zip --kind Acoustic --project "@my.project.txt" --output url "@my.dataset.test1.txt"
```

**Get status**

```
spx --% csr dataset status --dataset "@my.dataset.test1.txt" –wait
```

## Create a custom model

After all training data is uploaded, you can build a custom model using it. The custom model can include Pronunciation, Language, and Acoustic model adaptation, which is all done together.

### Combine dataset outputs into one file to use as input

Use PowerShell commands to combine the dataset details into one file.

```
Get-Content .\my.dataset.pm.txt -Raw | Add-Content -Path .\my.datasets.txt
Get-Content .\my.dataset.lm.txt -Raw | Add-Content -Path .\my.datasets.txt
```

### Create custom model

Use the `csr model create` command to create a custom model.

```
spx --% csr model create --project "@my.project.txt" --name githubsample_custom_model" --datasets "@my.datasets.txt" --output url "@my.model.txt"
```

**Get status**

```
spx --% csr model status --model "@my.model.txt" --wait
```

**Note:** When you create a custom model, you can also select a different version of the base model to use for the adaptations. If you don't provide a base model, the last base model that allows all kinds of adaptation (Language, Pronunciation, and Acoustic) is used. The following scripts show how to get a list of all available base models and then search for the specific base model to use for adaptation.

## Select a specific base model to use for adaptation

Use the `spx csr model list` command to list all base models and save the JSON output in a file. You can then search the file for the required base model, based on appropriate attributes.

**Note:** The last line of the output JSON might include a link  (@nextLink) for fetching additional results. If it does, fetch the additional results and merge the JSON results together before you perform your search.

### Get a list of base models and output it in a JSON file

```
spx --% csr model list --base models --output json baseModels.json
```

### Read the required base model details

Use the following PowerShell function to search for base models, based on the model's `displayName` and `locale`.

```
function get-baseModel-Json{
    param($name, $locale, $filePath)

    $baseModelJson = Get-Content $filePath -Raw | ConvertFrom-Json
    Write-Host 'Number of base models:' $baseModelJson.values.Count

    foreach ($value in $baseModelJson.values)
    {
        if($value.displayName -eq $name -and $value.locale -eq $locale) {
            return $value
        }
    }
}

$locale = "en-US"
$filePath = ".\baseModels.json"
$name = "20221013"

$bmJson = get-baseModel-Json -name $name -locale $locale -filePath $filePath

Write-Host $bmJson
```

## Evaluate the model

After your custom model is built, use the evaluation/testing dataset that you created earlier to run an evaluation. For the evaluation, the built custom model is generally compared with the base model that was used to build it. You can also compare two custom models. Use the `spx csr evaluation create` command to create an evaluation. In the following script, the evaluation compares the custom model that was just created with the base model from which it was built. For the custom model, the URL is fetched from the *my.model.txt* file. The base model URL is fetched from the JSON result produced by the preceding PowerShell script.

```
spx --% csr evaluation create --project "@my.project.txt" --name evaluation1 --model1 "@my.model.txt" --model2 "https://westus2.api.cognitive.microsoft.com/speechtotext/v3.0/models/base/modelGuidHere" --dataset "@my.dataset.test1.txt" --output url "@my.eval1.txt"
```

**Get status**

```
spx --% csr evaluation status --evaluation "@my.eval1.txt" --wait
```

## Publish the model

The last step is to publish the model and generate an endpoint that you can use in code to get the transcription result. Use the `csr endpoint create` command to publish the model.

```
spx --% csr endpoint create --project "@my.project.txt" --name "githubsample" --description "github sample model" --model "@my.model.txt" --output url "@my.endpoint.txt"
```

**Get status**

```
spx --% csr endpoint status --endpoint "@my.endpoint.txt" –wait
```
