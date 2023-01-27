<img src="https://lh3.googleusercontent.com/-Fq3nigRUo7U/VfaIPuJMjfI/AAAAAAAAALo/7oaLrrTBhnw/s1600/Fellow%2BOak%2BSquare%2BTransp.png" alt="fo-dicom logo" height="80" />

# Fellow Oak DICOM

[![NuGet](https://img.shields.io/nuget/v/fo-dicom.svg)](https://www.nuget.org/packages/fo-dicom/)
![build development](https://github.com/fo-dicom/fo-dicom/workflows/build/badge.svg?branch=development)
[![codecov](https://codecov.io/gh/fo-dicom/fo-dicom/branch/development/graph/badge.svg)](https://codecov.io/gh/fo-dicom/fo-dicom)

### License
This library is licensed under the [Microsoft Public License (MS-PL)](http://opensource.org/licenses/MS-PL). See [License.txt](License.txt) for more information.

### Features
* Targets .NET Standard 2.0
* DICOM dictionary version 2022b
* High-performance, fully asynchronous `async`/`await` API
* JPEG (including lossless), JPEG-LS, JPEG2000, and RLE image compression (via additional package)
* Supports very large datasets with content loading on demand
* Image rendering to System.Drawing.Bitmap or SixLabors.ImageSharp
* JSON and XML export/import
* Anonymization
* DICOM services
* Customize components via DI container

### Installation
Easiest is to obtain *fo-dicom* binaries from [NuGet](https://www.nuget.org/packages/fo-dicom/). This package reference the core *fo-dicom* assemblies for all Microsoft and Xamarin platforms.

### NuGet Packages
*Valid for version 5.0.0 and later*

Package | Description
------- | -----------
[fo-dicom](https://www.nuget.org/packages/fo-dicom/) | Core package containing parser, services and tools.
[fo-dicom.Imaging.Desktop](https://www.nuget.org/packages/fo-dicom.Imaging.Desktop/) | Library with referencte to System.Drawing, required for rendering into Bitmaps
[fo-dicom.Imaging.ImageSharp](https://www.nuget.org/packages/fo-dicom.Desktop/) | Library with reference to ImageSharp, can be used for platform independent rendering
[fo-dicom.NLog](https://www.nuget.org/packages/fo-dicom.NLog/) | .NET connector to enable *fo-dicom* logging with NLog
[fo-dicom.Codecs](https://www.nuget.org/packages/fo-dicom.Codecs/) | Cross-platform Dicom codecs for fo-dicom, developed by Efferent Health (https://github.com/Efferent-Health/fo-dicom.Codecs)


### Documentation
Documentation, including API documentation, is available via GitHub pages:
- documentation for the latest release for [fo-dicom 4](https://fo-dicom.github.io/stable/v4/index.html) and
  [fo-dicom 5](https://fo-dicom.github.io/stable/v5/index.html)
- documentation for the development version for [fo-dicom 4](https://fo-dicom.github.io/dev/v4/index.html) and
  [fo-dicom 5](https://fo-dicom.github.io/dev/v5/index.html)


### Usage Notes

#### Image rendering configuration
Out-of-the-box, *fo-dicom* defaults to an internal class *FellowOakDicom.Imaging.IImage*-style image rendering. To switch to Desktop-style or ImageSharp-style image rendering, you first have to add the nuget package you desire and then call:

    new DicomSetupBuilder()
        .RegisterServices(s => s.AddFellowOakDicom().AddImageManager<WinFormsImageManager>())
	.Build();

or

    new DicomSetupBuilder()
        .RegisterServices(s => s.AddFellowOakDicom().AddImageManager<ImageSharpImageManager>())
	.Build();

Then when rendering you can cast the IImage to the type by

    var image = new DicomImage("filename.dcm");
    var bitmap = image.RenderImage().As<Bitmap>();

or

    var image = new DicomImage("filename.dcm");
    var sharpimage = image.RenderImage().AsSharpImage();

#### Logging configuration
By default, logging defaults to the no-op `NullLogerManager`. There are several logmanagers configurable within `DicomSetupBuilder` like

    s.AddLogManager<ConsoleLogManager>()  // or ...
    s.AddLogManager<NLogManager>()   // or ...


    LogManager.SetImplementation(ConsoleLogManager.Instance);  // or ...
    LogManager.SetImplementation(NLogManager.Instance);        // or ...

### Sample applications
There are a number of simple sample applications that use *fo-dicom* available in separate repository [here](https://github.com/fo-dicom/fo-dicom-samples). These also include the samples
that were previously included in the *Examples* sub-folder of the VS solutions.

### Examples

#### File Operations
```csharp
var file = DicomFile.Open(@"test.dcm");             // Alt 1
var file = await DicomFile.OpenAsync(@"test.dcm");  // Alt 2

var patientid = file.Dataset.GetString(DicomTag.PatientID);

file.Dataset.AddOrUpdate(DicomTag.PatientName, "DOE^JOHN");

// creates a new instance of DicomFile
var newFile = file.Clone(DicomTransferSyntax.JPEGProcess14SV1);

file.Save(@"output.dcm");             // Alt 1
await file.SaveAsync(@"output.dcm");  // Alt 2
```

#### Render Image to JPEG
```csharp
var image = new DicomImage(@"test.dcm");
image.RenderImage().AsBitmap().Save(@"test.jpg");                     // Windows Forms

```

#### C-Store SCU
```csharp
var client = DicomClientFactory.Create("127.0.0.1", 12345, false, "SCU", "ANY-SCP");
await client.AddRequestAsync(new DicomCStoreRequest(@"test.dcm"));
await client.SendAsync();
```

#### C-Echo SCU/SCP
```csharp
var server = new DicomServer<DicomCEchoProvider>(12345);

var client = DicomClientFactory.Create("127.0.0.1", 12345, false, "SCU", "ANY-SCP");
client.NegotiateAsyncOps();
for (int i = 0; i < 10; i++)
    await client.AddRequestAsync(new DicomCEchoRequest());
await client.SendAsync();
```

#### C-Find SCU
```csharp
var cfind = DicomCFindRequest.CreateStudyQuery(patientId: "12345");
cfind.OnResponseReceived = (DicomCFindRequest rq, DicomCFindResponse rp) => {
	Console.WriteLine("Study UID: {0}", rp.Dataset.Get<string>(DicomTag.StudyInstanceUID));
};

var client = DicomClientFactory.Create("127.0.0.1", 11112, false, "SCU-AE", "SCP-AE");
await client.AddRequestAsync(cfind);
await client.SendAsync();
```

#### C-Move SCU
```csharp
var cmove = new DicomCMoveRequest("DEST-AE", studyInstanceUid);

var client = DicomClientFactory.Create("127.0.0.1", 11112, false, "SCU-AE", "SCP-AE");
await client.AddRequestAsync(cmove);
await client.SendAsync();
```

#### N-Action SCU
```csharp
// It is better to increase 'associationLingerTimeoutInMs' default is 50 ms, which may not be
// be sufficient
var dicomClient = DicomClientFactory.Create("127.0.0.1", 12345, false, "SCU-AE", "SCP-AE",
DicomClientDefaults.DefaultAssociationRequestTimeoutInMs, DicomClientDefaults.DefaultAssociationReleaseTimeoutInMs,5000);
var txnUid = DicomUIDGenerator.GenerateDerivedFromUUID().UID;
var nActionDicomDataSet = new DicomDataset
{
    { DicomTag.TransactionUID,  txnUid }
};
var dicomRefSopSequence = new DicomSequence(DicomTag.ReferencedSOPSequence);
var seqItem = new DicomDataset()
{
    { DicomTag.ReferencedSOPClassUID, "1.2.840.10008.5.1.4.1.1.1" },
    { DicomTag.ReferencedSOPInstanceUID, "1.3.46.670589.30.2273540226.4.54" }
};
dicomRefSopSequence.Items.Add(seqItem);
nActionDicomDataSet.Add(dicomRefSopSequence);
var nActionRequest = new DicomNActionRequest(DicomUID.StorageCommitmentPushModelSOPClass,
                DicomUID.StorageCommitmentPushModelSOPInstance, 1)
{
    Dataset = nActionDicomDataSet,
    OnResponseReceived = (DicomNActionRequest request, DicomNActionResponse response) =>
    {
        Console.WriteLine("NActionResponseHandler, response status:{0}", response.Status);
    },
};
await dicomClient.AddRequestAsync(nActionRequest);
dicomClient.OnNEventReportRequest = OnNEventReportRequest;
await dicomClient.SendAsync();

private static Task<DicomNEventReportResponse> OnNEventReportRequest(DicomNEventReportRequest request)
{
    var refSopSequence = request.Dataset.GetSequence(DicomTag.ReferencedSOPSequence);
    foreach(var item in refSopSequence.Items)
    {
        Console.WriteLine("SOP Class UID: {0}", item.GetString(DicomTag.ReferencedSOPClassUID));
        Console.WriteLine("SOP Instance UID: {0}", item.GetString(DicomTag.ReferencedSOPInstanceUID));
    }
    return Task.FromResult(new DicomNEventReportResponse(request, DicomStatus.Success));
}
```

#### C-ECHO with advanced DICOM client connection: manual control over TCP connection and DICOM association
```csharp
var cancellationToken = CancellationToken.None;
// Alternatively, inject IDicomServerFactory via dependency injection instead of using this static method
using var server = DicomServerFactory.Create<DicomCEchoProvider>(12345);

var connectionRequest = new AdvancedDicomClientConnectionRequest
{
    NetworkStreamCreationOptions = new NetworkStreamCreationOptions
    {
        Host = "127.0.0.1",
        Port = server.Port,
    }
};

// Alternatively, inject IAdvancedDicomClientConnectionFactory via dependency injection instead of using this static method
using var connection = await AdvancedDicomClientConnectionFactory.OpenConnectionAsync(connectionRequest, cancellationToken);

var associationRequest = new AdvancedDicomClientAssociationRequest
{
    CallingAE = "EchoSCU",
    CalledAE = "EchoSCP"
};

var cEchoRequest = new DicomCEchoRequest();

using var association = await connection.OpenAssociationAsync(associationRequest, cancellationToken);
try
{
    DicomCEchoResponse cEchoResponse = await association.SendCEchoRequestAsync(cEchoRequest, cancellationToken).ConfigureAwait(false);

    Console.WriteLine(cEchoResponse.Status);
}
finally
{
    await association.ReleaseAsync(cancellationToken);
}
```

## Contributors ✨

Thanks go to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/anders9ustafsson"><img src="https://avatars.githubusercontent.com/u/6515030?v=4?s=100" width="100px;" alt="Anders Gustafsson"/><br /><sub><b>Anders Gustafsson</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=anders9ustafsson" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/amoerie"><img src="https://avatars.githubusercontent.com/u/2063584?v=4?s=100" width="100px;" alt="Alexander Moerman"/><br /><sub><b>Alexander Moerman</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=amoerie" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://www.linkedin.com/in/cdillion"><img src="https://avatars.githubusercontent.com/u/280475?v=4?s=100" width="100px;" alt="Colby Dillion"/><br /><sub><b>Colby Dillion</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=rcd" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/gofal"><img src="https://avatars.githubusercontent.com/u/6769246?v=4?s=100" width="100px;" alt="Reinhard Gruber"/><br /><sub><b>Reinhard Gruber</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=gofal" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/mia-san"><img src="https://avatars.githubusercontent.com/u/8837998?v=4?s=100" width="100px;" alt="mia-san"/><br /><sub><b>mia-san</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=mia-san" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/mrbean-bremen"><img src="https://avatars.githubusercontent.com/u/4623701?v=4?s=100" width="100px;" alt="mrbean-bremen"/><br /><sub><b>mrbean-bremen</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=mrbean-bremen" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/rickardraysearch"><img src="https://avatars.githubusercontent.com/u/1135306?v=4?s=100" width="100px;" alt="Rickard Holmberg"/><br /><sub><b>Rickard Holmberg</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=rickardraysearch" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://www.medicalit.com.au/"><img src="https://avatars.githubusercontent.com/u/1314477?v=4?s=100" width="100px;" alt="Ian Yates"/><br /><sub><b>Ian Yates</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=IanYates" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/BobSter3000"><img src="https://avatars.githubusercontent.com/u/9662532?v=4?s=100" width="100px;" alt="Bob Woods"/><br /><sub><b>Bob Woods</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=BobSter3000" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/shivakumar-ms"><img src="https://avatars.githubusercontent.com/u/94023192?v=4?s=100" width="100px;" alt="Shiva Kumar"/><br /><sub><b>Shiva Kumar</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=shivakumar-ms" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/pengchen0692"><img src="https://avatars.githubusercontent.com/u/67345894?v=4?s=100" width="100px;" alt="Peng Chen"/><br /><sub><b>Peng Chen</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=pengchen0692" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/MaherJendoubi"><img src="https://avatars.githubusercontent.com/u/1798510?v=4?s=100" width="100px;" alt="Maher Jendoubi"/><br /><sub><b>Maher Jendoubi</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=MaherJendoubi" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="http://www.distributedmedical.com/"><img src="https://avatars.githubusercontent.com/u/47624589?v=4?s=100" width="100px;" alt="Björn Lundmark"/><br /><sub><b>Björn Lundmark</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=bjorn-malmo" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/mdubey82"><img src="https://avatars.githubusercontent.com/u/286282?v=4?s=100" width="100px;" alt="Mahesh"/><br /><sub><b>Mahesh</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=mdubey82" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://www.nebrastech.com/"><img src="https://avatars.githubusercontent.com/u/2395913?v=4?s=100" width="100px;" alt="Hesham Desouky"/><br /><sub><b>Hesham Desouky</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=hdesouky" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://medium.com/@smithasaligrama"><img src="https://avatars.githubusercontent.com/u/6980724?v=4?s=100" width="100px;" alt="Smitha Saligrama"/><br /><sub><b>Smitha Saligrama</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=smithago" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://www.linkedin.com/in/zaidsafadi/"><img src="https://avatars.githubusercontent.com/u/10813109?v=4?s=100" width="100px;" alt="Zaid Safadi"/><br /><sub><b>Zaid Safadi</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=Zaid-Safadi" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/javier-alvarez"><img src="https://avatars.githubusercontent.com/u/707304?v=4?s=100" width="100px;" alt="Javier"/><br /><sub><b>Javier</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=javier-alvarez" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://www.linkedin.com/in/sudheesh-p-s-429a8246/"><img src="https://avatars.githubusercontent.com/u/40300928?v=4?s=100" width="100px;" alt="Sudheesh Subramannian"/><br /><sub><b>Sudheesh Subramannian</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=sudheeshps" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://www.m-bit.nl/"><img src="https://avatars.githubusercontent.com/u/3763706?v=4?s=100" width="100px;" alt="Michael M"/><br /><sub><b>Michael M</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=MichaelM223" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/idubnori"><img src="https://avatars.githubusercontent.com/u/22805312?v=4?s=100" width="100px;" alt="idubnori"/><br /><sub><b>idubnori</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=idubnori" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://pe.linkedin.com/in/jaimeolivaresf"><img src="https://avatars.githubusercontent.com/u/6194083?v=4?s=100" width="100px;" alt="Jaime Olivares"/><br /><sub><b>Jaime Olivares</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=jaime-olivares" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/abdullah248"><img src="https://avatars.githubusercontent.com/u/31964511?v=4?s=100" width="100px;" alt="Abdullah Islam"/><br /><sub><b>Abdullah Islam</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=abdullah248" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/mocsharp"><img src="https://avatars.githubusercontent.com/u/4236590?v=4?s=100" width="100px;" alt="Victor Chang"/><br /><sub><b>Victor Chang</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=mocsharp" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/apps/dependabot"><img src="https://avatars.githubusercontent.com/in/29110?v=4?s=100" width="100px;" alt="dependabot[bot]"/><br /><sub><b>dependabot[bot]</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=dependabot[bot]" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/pvrobays"><img src="https://avatars.githubusercontent.com/u/29975312?v=4?s=100" width="100px;" alt="Pieter-Jan Van Robays"/><br /><sub><b>Pieter-Jan Van Robays</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=pvrobays" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://kira-96.github.io/"><img src="https://avatars.githubusercontent.com/u/22876709?v=4?s=100" width="100px;" alt="kirakira"/><br /><sub><b>kirakira</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=kira-96" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/JustSomeHack"><img src="https://avatars.githubusercontent.com/u/4336597?v=4?s=100" width="100px;" alt="JustSomeHack"/><br /><sub><b>JustSomeHack</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=JustSomeHack" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/lste"><img src="https://avatars.githubusercontent.com/u/3227014?v=4?s=100" width="100px;" alt="lste"/><br /><sub><b>lste</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=lste" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/gladiatus55"><img src="https://avatars.githubusercontent.com/u/25867804?v=4?s=100" width="100px;" alt="Martin Nguyen"/><br /><sub><b>Martin Nguyen</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=gladiatus55" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/atifc"><img src="https://avatars.githubusercontent.com/u/26510982?v=4?s=100" width="100px;" alt="atifc"/><br /><sub><b>atifc</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=atifc" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/sisinio"><img src="https://avatars.githubusercontent.com/u/5585987?v=4?s=100" width="100px;" alt="sisinio"/><br /><sub><b>sisinio</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=sisinio" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/dalingrin"><img src="https://avatars.githubusercontent.com/u/71467?v=4?s=100" width="100px;" alt="Erik Hardesty"/><br /><sub><b>Erik Hardesty</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=dalingrin" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/JoostJM"><img src="https://avatars.githubusercontent.com/u/18026947?v=4?s=100" width="100px;" alt="Joost van Griethuysen"/><br /><sub><b>Joost van Griethuysen</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=JoostJM" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/Thunderstriker"><img src="https://avatars.githubusercontent.com/u/279965?v=4?s=100" width="100px;" alt="Thunderstriker"/><br /><sub><b>Thunderstriker</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=Thunderstriker" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/npnelson"><img src="https://avatars.githubusercontent.com/u/2856705?v=4?s=100" width="100px;" alt="Nicholas P Nelson"/><br /><sub><b>Nicholas P Nelson</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=npnelson" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/HSGorman"><img src="https://avatars.githubusercontent.com/u/7083436?v=4?s=100" width="100px;" alt="HSGorman"/><br /><sub><b>HSGorman</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=HSGorman" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/MacL3an"><img src="https://avatars.githubusercontent.com/u/2429736?v=4?s=100" width="100px;" alt="Håkan MacLean"/><br /><sub><b>Håkan MacLean</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=MacL3an" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/HwangTaehyun"><img src="https://avatars.githubusercontent.com/u/42465090?v=4?s=100" width="100px;" alt="Taehyun Hwang"/><br /><sub><b>Taehyun Hwang</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=HwangTaehyun" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/MichaelOvens"><img src="https://avatars.githubusercontent.com/u/36658325?v=4?s=100" width="100px;" alt="Michael Ovens"/><br /><sub><b>Michael Ovens</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=MichaelOvens" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/Johannes-sg"><img src="https://avatars.githubusercontent.com/u/60503649?v=4?s=100" width="100px;" alt="Johannes Hirschmann"/><br /><sub><b>Johannes Hirschmann</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=Johannes-sg" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/nutzlastfan"><img src="https://avatars.githubusercontent.com/u/25078990?v=4?s=100" width="100px;" alt="Denny Spiegelberg"/><br /><sub><b>Denny Spiegelberg</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=nutzlastfan" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/devsko"><img src="https://avatars.githubusercontent.com/u/12471105?v=4?s=100" width="100px;" alt="devsko"/><br /><sub><b>devsko</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=devsko" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/CDarbonne"><img src="https://avatars.githubusercontent.com/u/9934740?v=4?s=100" width="100px;" alt="Chris Darbonne"/><br /><sub><b>Chris Darbonne</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=CDarbonne" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://profoundmedical.com/"><img src="https://avatars.githubusercontent.com/u/56166202?v=4?s=100" width="100px;" alt="ProfoundMedicalDeveloper"/><br /><sub><b>ProfoundMedicalDeveloper</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=ProfoundMedicalDeveloper" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/nodonnell"><img src="https://avatars.githubusercontent.com/u/7482728?v=4?s=100" width="100px;" alt="Niall"/><br /><sub><b>Niall</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=nodonnell" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/Cregganbane"><img src="https://avatars.githubusercontent.com/u/28092624?v=4?s=100" width="100px;" alt="Laurence"/><br /><sub><b>Laurence</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=Cregganbane" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/werwolfby"><img src="https://avatars.githubusercontent.com/u/705754?v=4?s=100" width="100px;" alt="Alexander Puzynia"/><br /><sub><b>Alexander Puzynia</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=werwolfby" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="http://www.ryanmelena.com/"><img src="https://avatars.githubusercontent.com/u/7570707?v=4?s=100" width="100px;" alt="Ryan Melena"/><br /><sub><b>Ryan Melena</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=RyanMelenaNoesis" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="http://www.saratow.biz/"><img src="https://avatars.githubusercontent.com/u/4418316?v=4?s=100" width="100px;" alt="Alexander Saratow"/><br /><sub><b>Alexander Saratow</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=swalex" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/akojo"><img src="https://avatars.githubusercontent.com/u/100779?v=4?s=100" width="100px;" alt="Atte Kojo"/><br /><sub><b>Atte Kojo</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=akojo" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/Amdhj22"><img src="https://avatars.githubusercontent.com/u/67995652?v=4?s=100" width="100px;" alt="Andy Eum"/><br /><sub><b>Andy Eum</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=Amdhj22" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://www.linkedin.com/in/john-cupitt-7a422b151/"><img src="https://avatars.githubusercontent.com/u/580843?v=4?s=100" width="100px;" alt="John Cupitt"/><br /><sub><b>John Cupitt</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=jcupitt" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/magla42"><img src="https://avatars.githubusercontent.com/u/10060945?v=4?s=100" width="100px;" alt="Magnus Larsson"/><br /><sub><b>Magnus Larsson</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=magla42" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="http://www.ceejaycee.net/"><img src="https://avatars.githubusercontent.com/u/5764748?v=4?s=100" width="100px;" alt="Chris Conway"/><br /><sub><b>Chris Conway</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=CeeJayCee" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://about.me/jsutherland"><img src="https://avatars.githubusercontent.com/u/9466?v=4?s=100" width="100px;" alt="James A Sutherland"/><br /><sub><b>James A Sutherland</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=jas88" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/jovinson-ms"><img src="https://avatars.githubusercontent.com/u/88204686?v=4?s=100" width="100px;" alt="Josiah Vinson"/><br /><sub><b>Josiah Vinson</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=jovinson-ms" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://www.linkedin.com/in/wsugarman"><img src="https://avatars.githubusercontent.com/u/2955179?v=4?s=100" width="100px;" alt="Will Sugarman"/><br /><sub><b>Will Sugarman</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=wsugarman" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/fredrikcarlbom"><img src="https://avatars.githubusercontent.com/u/45489326?v=4?s=100" width="100px;" alt="Fredrik Carlbom"/><br /><sub><b>Fredrik Carlbom</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=fredrikcarlbom" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/marber1973"><img src="https://avatars.githubusercontent.com/u/79089839?v=4?s=100" width="100px;" alt="Marc Berreth"/><br /><sub><b>Marc Berreth</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=marber1973" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/gasupidupi"><img src="https://avatars.githubusercontent.com/u/43344400?v=4?s=100" width="100px;" alt="gasupidupi"/><br /><sub><b>gasupidupi</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=gasupidupi" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/EthicsGradient"><img src="https://avatars.githubusercontent.com/u/3604021?v=4?s=100" width="100px;" alt="James"/><br /><sub><b>James</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=EthicsGradient" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/OuReact"><img src="https://avatars.githubusercontent.com/u/25136594?v=4?s=100" width="100px;" alt="OuReact"/><br /><sub><b>OuReact</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=OuReact" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/shazha"><img src="https://avatars.githubusercontent.com/u/4388678?v=4?s=100" width="100px;" alt="shazha"/><br /><sub><b>shazha</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=shazha" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/feliciwithme"><img src="https://avatars.githubusercontent.com/u/15883404?v=4?s=100" width="100px;" alt="feliciwithme"/><br /><sub><b>feliciwithme</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=feliciwithme" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/paul81"><img src="https://avatars.githubusercontent.com/u/35716868?v=4?s=100" width="100px;" alt="paul81"/><br /><sub><b>paul81</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=paul81" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/vbaderks"><img src="https://avatars.githubusercontent.com/u/4932528?v=4?s=100" width="100px;" alt="Victor Derks"/><br /><sub><b>Victor Derks</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=vbaderks" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/jwbruce93"><img src="https://avatars.githubusercontent.com/u/34671357?v=4?s=100" width="100px;" alt="Jamie Bruce"/><br /><sub><b>Jamie Bruce</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=jwbruce93" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/amosonn"><img src="https://avatars.githubusercontent.com/u/3142573?v=4?s=100" width="100px;" alt="amosonn"/><br /><sub><b>amosonn</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=amosonn" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/zcr01"><img src="https://avatars.githubusercontent.com/u/18120426?v=4?s=100" width="100px;" alt="zcr01"/><br /><sub><b>zcr01</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=zcr01" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/do0om"><img src="https://avatars.githubusercontent.com/u/623156?v=4?s=100" width="100px;" alt="do0om"/><br /><sub><b>do0om</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=do0om" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://gitter.im/"><img src="https://avatars.githubusercontent.com/u/8518239?v=4?s=100" width="100px;" alt="The Gitter Badger"/><br /><sub><b>The Gitter Badger</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=gitter-badger" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="16.66%"><a href="https://www.linkedin.com/in/chafey"><img src="https://avatars.githubusercontent.com/u/1268698?v=4?s=100" width="100px;" alt="Chris Hafey"/><br /><sub><b>Chris Hafey</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=chafey" title="Code">💻</a></td>
      <td align="center" valign="top" width="16.66%"><a href="https://github.com/captainstark"><img src="https://avatars.githubusercontent.com/u/5644386?v=4?s=100" width="100px;" alt="captainstark"/><br /><sub><b>captainstark</b></sub></a><br /><a href="https://github.com/fo-dicom/fo-dicom/commits?author=captainstark" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://allcontributors.org) specification.
Contributions of any kind are welcome!

### New to DICOM?

If you are new to DICOM, then take a look at the DICOM tutorial of Saravanan Subramanian:
https://saravanansubramanian.com/dicomtutorials/
The author is also using fo-dicom in some code samples.
