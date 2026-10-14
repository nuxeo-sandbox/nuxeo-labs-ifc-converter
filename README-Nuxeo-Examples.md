Nuxeo Examples
========================

Examples of using the ifc-converter, using Nuxeo Studio configuration.

### • Set the thumbnail of an .ifc Document (Nuxeo Studio configuration)

In this example, we want the thumbnail (PNG, 3d) to be cacluated automatically when file:content is modified
(so, creation/modification). Now, calculating the thumbnail may take time (usually fast, < 1 second, but
could take dozens of seconds on complex ifc, or just a longer-than) usual Nuxeo-<>ifc-converter containers
communication). So:<br/>
=> It should not be done synchronously (in aboutToCreate, or beforeDocumentModification).<br/>
=> **The chain should be called in an asynchronous event for "Document modified"**
   
**But** there is a "async work flow" issue. Basically, when file:content is modified (creation/modif), 2 async
works are started, each in its own thread:
* this event,
* and the `updateThumbListener` work from Nuxeo.

The later removes the `Thumbnail` facet if it can't generate a thumbnail. So, depending on CPU, OS threads etc.
it erases a Thumbnail calculated in the event.

1. **Event Handler**

* Event: `DocumentModified`
* Event Handler Activation:
  * Current document has one of the types: `File` (or whatever custom document type you created)
  * Current documetnt is: Regular Document
* Chain (JavaScript): `FileModified_IFC_SetThumbnail` (or whatever name you want) 

2. **JS Automation Chain**

```js
/* (see explanation above)
   Acting only once the facet is gone guarantees
    -> that we run after Nuxeo Thumbnail Listener
    -> and that addFacet is a real add, not a no-op.
     
     It is also the loop guard: our own save re-adds the facet, so the next documentModified
     exits at the beginning.
*/
function run(input, params) {
  
  var blob, png3d;
  
  // Immediate leave if the facet is here (either set by Nuxeo or by this same code)
  if (input.hasFacet("Thumbnail")) {
    return input;
  }

  blob = input["file:content"];
  if (!blob || blob.getMimeType() !== "model/ifc") {
    return input;
  }

  png3d = Blob.RunConverter(blob, { 'converter': "ifc-to-png", 'parameters': 'view=3d\nsize=1024' });
  if (!png3d) {
    return input;
  }
  
  Console.log("FileModified_IFC_SetThumbnail...");

  input.addFacet("Thumbnail"); // Faster than calling Document.AddFacet
  input["thumb:thumbnail"] = png3d;
  // The following trigers again "document modified" but we will not enter an infinite loop since
  // the facet is tested at the very beginning
  input = Document.Save(input, {});

  return input;

}
```

> [!IMPORTANT]
> Using a JS Automation chain is fine for demos and test. In production, you should write a Java plugin that
> overrides the default Thumbnail Factory: Handles the thumbnail if the file is .ifc, call the default code
> if not (pdf, docx, ...)

### • Display the Text Content in the PDF Viewer - The Preview if Available

#### 1. Add a CommandLine Converter

`soffice` converts nicely from .ipc to .pdf. At runtime, Nuxeo uses the Mime type to choose a converter (so make sure you contributed the correct Mime Type, see README)

```xml
<!-- soffice can convert to pdf
     Based on the mime-typ, nuxeo-document-preview will get the conversion -->
<extension target="org.nuxeo.ecm.core.convert.service.ConversionServiceImpl" point="converter">
  <converter name="ifc2pdf" class="org.nuxeo.ecm.platform.convert.plugins.LibreOfficeConverter">
    <sourceMimeType>model/ifc</sourceMimeType>
    <destinationMimeType>application/pdf</destinationMimeType>
    <parameters>
      <parameter name="CommandLineName">soffice</parameter>
      <parameter name="format">pdf</parameter>
    </parameters>
  </converter>
</extension>
```

#### 2. Example of use by overriding the `nuxeo-file-view-layout

Here, we reuse the thumbnail created above. Basicaly, if current document is an ifc and has a thumbnail, we use it, else we fall back to default viewer.

You may want something more complex. Typically, you could add a "IFC" facet (with whatever "ifc" schema and field you need), ot maybe you have a dedicated `IFC` dcument type and you generate the `GLB` converstion to display it as a preview, etc.

```html
<dom-module id="nuxeo-file-view-layout">
  <template>
    <style include="nuxeo-styles">
      :host {
        @apply --paper-card;
      }
      nuxeo-image-viewer {
        width: 100%;
        height: 100%;
        min-height: var(--nuxeo-viewer-min-height, 60vh);
      }
    </style>

    <template is="dom-if" if="[[_isIFCWithPreview(document)]]">
      <nuxeo-image-viewer
        src="[[_getIFCPreviewUrl(document)]]"
        controls
        responsive
        alt$="[[document.title]]"
      ></nuxeo-image-viewer>
    </template>

    <template is="dom-if" if="[[!_isIFCWithPreview(document)]]">
      <nuxeo-document-viewer role="widget" document="[[document]]"></nuxeo-document-viewer>
    </template>
  </template>

  <script>
    Polymer({
      is: 'nuxeo-file-view-layout',
      behaviors: [Nuxeo.LayoutBehavior],
      properties: {
        /**
         * @doctype File
         */
        document: Object,
      },

      _isIFCWithPreview: function(document) {
        if(document && document.properties && document.properties['file:content'] && document.properties['file:content']["mime-type"] === 'model/ifc') {
          // In this example use thumb
          return document.properties["thumb:thumbnail"] ? true : false;
        }
        return false;
      },

      _getIFCPreviewUrl: function(document) {
        return document.properties["thumb:thumbnail"].blobUrl;
      }
    });
  </script>
</dom-module>
```