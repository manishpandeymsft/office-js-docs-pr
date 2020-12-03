---
title: Insert and delete slides in a PowerPoint presentation
description: 'Learn how to insert slides from one presentation into another and how to delete slides.'
ms.date: 12/04/2020
localization_priority: Normal
---

# Insert and delete slides in a PowerPoint presentation

A PowerPoint add-in can insert slides from one presentation (in base64 format) into the current presentation by using PowerPoint's application-specific JavaScript library. You can control whether the inserted slides keep the formatting of the source presentation or the formatting of the target presentation. You can also delete slides from the presentation.

There are two major steps to inserting slides from one presentation into another.

1. Convert the source presentation file (.pptx) into a base64-formatted file.
1. Use the `insertSlidesFromBase64` method to insert one or more slides from the base64 file into the current presentation.

## Convert the source presentation to base64

There are many ways to convert a file to base64. Which programming language and library you use, and whether to convert on the server-side of your add-in or the client-side is determined by your scenario. Most commonly, you'll do the conversion in JavaScript on the client-side by using a [FileReader](https://developer.mozilla.org/en-US/docs/Web/API/FileReader) object. The following is an example.

1. Begin by getting a reference to the source PowerPoint file. In this example, we will use an `<input>` control of type `file` to prompt the user to choose a file. Add the following markup to the add-in page.

    ```html
    <section>
        <p>Select a PowerPoint presentation from which to insert slides</p>
        <form>
            <input type="file" id="file" />
        </form>
    </section>
    ```

    This markup adds the UI in the following screenshot to the page:

    ![Screenshot showing an HTML file type input control preceded by an instructional sentence reading "Select a PowerPoint presentation from which to insert slides". The control consists of a button labelled "Choose file" followed by the sentence "No file chosen".](../images/powerpoint-html-file-input-control.png)

2. Add the following code to the add-in's JavaScript to assign a function to the input control's `change` event. (You create the `storeFileAsBase64` in the next step.)

    ```javascript
    $("#file").change(storeFileAsBase64);
    ```

3. Add the following code. About this code, note the following:

    - The `reader.readAsDataURL` method converts the file to base64 and stores it in the `reader.result` property. When the method completes, it triggers the `onload` event handler.
    - The `onload` event handler trims metadata off of the encoded file and stores the encoded string in a global variable.
    - The base64-encoded string is stored globally because it will be read by another function that you create in a later step.

    ```javascript
    let chosenFileBase64;

    async function storeFileAsBase64() {
        const reader = new FileReader();

        reader.onload = async (event) => {
            const startIndex = reader.result.toString().indexOf("base64,");
            const copyBase64 = reader.result.toString().substr(startIndex + 7);

            chosenFileBase64 = copyBase64;
        };

        const myFile = document.getElementById("file") as HTMLInputElement;
        reader.readAsDataURL(myFile.files[0]);
    }
    ```

## Insert slides with insertSlidesFromBase64

You insert slides from another PowerPoint presentation into the current presentation with the [Presentation.insertSlidesFromBase64](/javascript/api/powerpoint/powerpoint.presentation#insertslidesfrombase64-base64file--options-) method. The following is a simple example in which all of the slides from the source presentation are inserted at the beginning of the current presentation and the inserted slides keep the formatting of the source file. Note that `chosenFileBase64` is a global variable that holds a base64-encoded version of a PowerPoint presentation file.

```javascript
async function insertAllSlides() {
  await PowerPoint.run(async function(context) {
    context.presentation.insertSlidesFromBase64(chosenFileBase64);
    await context.sync();
  });
}
```

You can control some aspects of the insertion result, including where the slides are inserted and whether they get the source or target formatting (and more), by passing a [InsertSlideOptions](/javascript/api/powerpoint/powerpoint.insertslideoptions?view=powerpoint-js-preview) object as a second parameter to `insertSlidesFromBase64`. The following is an example. About this code, note:

- There are two possible values for the `formatting` property: "UseDestinationTheme" and "KeepSourceFormatting". Optionally, you can use the full name of the `InsertSlideFormatting` enum, such as `PowerPoint.InsertSlideFormatting.useDestinationTheme`.
- The function will insert the slides from the source presentation immediately after the slide specified by the `targetSlideId` property. The value of this property is a string of one of three possible forms: ***nnn*#**, **#*mmmmmmmmm***, or ***nnn*#*mmmmmmmmm***, where *nnn* is the slide's ID (typically 3 digits) and *mmmmmmmmm* is the slide's creation ID (typically, 9 digits). Some examples are `267#763315295`, `267#`, and `#763315295`.

```javascript
async function insertSlidesDestinationFormatting() {
  await PowerPoint.run(async function(context) {
    context.presentation
    .insertSlidesFromBase64(chosenFileBase64,
                            {
                                formatting: "UseDestinationTheme",
                                targetSlideId: "267#"
                            }
                          );
    await context.sync();
  });
}
```

Of course, you typically won't know at coding time the ID or creation ID of the target slide. More commonly, an add-in will ask users to select the target slide. The following steps show how to get the ***nnn*#** ID of the currently selected slide and use it as the target slide.

1. Create a function that gets the ID of the currently selected slide by using the `getSelectedDataAsync` method of the Common JavaScript APIs. The following is an example. Note that the call to `getSelectedDataAsync` is embedded in a Promise-returning function. It is a good practice to do this because the Promise-returning function is easy to call (and optionally await) inside the `run()` method of an application-specific API. By awaiting the Promise-returning function, you are assured that the callback function of the Common API method has completed before the next line in the `run()` method executes.

 
    ```javascript
    function getSelectedSlideID() {
      return new OfficeExtension.Promise<string>(function (resolve, reject) {
        Office.context.document.getSelectedDataAsync(Office.CoercionType.SlideRange, function (asyncResult) {
          try {
            if (asyncResult.status === Office.AsyncResultStatus.Failed) {
              reject(console.error(asyncResult.error.message));
            } else {
              resolve(asyncResult.value.slides[0].id);
            }
          }
          catch (error) {
            reject(console.log(error));
          }
        });
      })
    }
    ```

1. Call your new function inside the `PowerPoint.run()` of the main function and pass the ID that it returns (concatenated with the "#" symbol) as the value of the `targetSlideId` property of the `InsertSlideOptions` parameter. The following is an example.

    ```javascript
    async function insertAfterSelectedSlide() {
    await PowerPoint.run(async function(context) {

        const selectedSlideID = await getSelectedSlideID();

        context.presentation.insertSlidesFromBase64(chosenFileBase64, {
          formatting: PowerPoint.InsertSlideFormatting.useDestinationTheme,
          targetSlideId: selectedSlideID + "#"
        });

        await context.sync();
      });
    }
    ```

### Selecting which slides to insert



## Delete slides
