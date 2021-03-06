<properties 
	pageTitle="Continous export of telemetry from Application Insights" 
	description="Export diagnostic and usage data to storage in Microsoft Azure, and download it from there." 
	services="application-insights" 
    documentationCenter=""
	authors="alancameronwills" 
	manager="douge"/>

<tags 
	ms.service="application-insights" 
	ms.workload="tbd" 
	ms.tgt_pltfrm="ibiza" 
	ms.devlang="na" 
	ms.topic="article" 
	ms.date="07/08/2015" 
	ms.author="awills"/>
 
# Export telemetry from Application Insights

Want to do some customised analysis on your telemetry? Or maybe you'd like an email alert on events with specific properties? Continuous Export is ideal for this. The events you see in the Application Insights portal can be exported to storage in Microsoft Azure in JSON format. From there you can download your data and write whatever code you need to process it.  

Continuous Export is available in the free trial period and on the [Standard and Premium pricing plans](http://azure.microsoft.com/pricing/details/application-insights/).

(If you just want to do a [one-off export](app-insights-metrics-explorer.md#export-to-excel) of what you see on a metrics or search blade, click Export at the top of the blade.)

## <a name="setup"></a> Set up Continuous Export

On your application's Overview blade in the Application Insights portal, open Continuous Export: 

![Scroll down and click Continuous Export](./media/app-insights-export-telemetry/01-export.png)

Add an export, and choose an [Azure storage account](../storage-introduction.md) where you want to put the data:

![Click Add, Export Destination, Storage account, and then either create a new store or choose an existing store](./media/app-insights-export-telemetry/02-add.png)

Choose the event types you'd like to export:

![Click Choose event types](./media/app-insights-export-telemetry/03-types.png)


Once you've created your export, it starts going. (You only get data that arrives after you create the export.)


If you want to change the event types later, just edit the export:

![Click Choose event types](./media/app-insights-export-telemetry/05-edit.png)

To stop the stream, click Disable. When you click Enable again, the stream will restart with new data. You won't get the data that arrived in the portal while export was disabled.

To stop the stream permanently, delete the export. Doing so doesn't delete your data from storage.

#### Can't add or change an export?

* To add or change exports, you need Owner, Contributor or Application Insights Contributor access rights. [Learn about roles][roles].

## <a name="analyze"></a> What events do you get?

The exported data is the raw telemetry we receive from your application, except: 

* Web test results aren't currently included. 
* We add location data which we calculate from the client IP address.  

Calculated metrics are not included. For example, we don't export average CPU utilisation, but we do export the raw telemetry from which the average is computed.

## <a name="get"></a> Inspect the data

When you open your blob store with a tool such as [Server Explorer](http://msdn.microsoft.com/library/azure/ff683677.aspx), you'll see a container with a set of blob files. The URI of each file is application-id/telemetry-type/date/time. 

![Inspect the blob store with a suitable tool](./media/app-insights-export-telemetry/04-data.png)

The date and time are UTC and are when the telemetry was deposited in the store - not the time it was generated. So if you write code to download the data, it can move linearly through the data.



## <a name="format"></a> Data format

* Each blob is a text file that contains multiple '\n'-separated rows.
* Each row is an unformatted JSON document. If you want to sit and stare at it, try a viewer such as Notepad++ with the JSON plug-in:

![View the telemetry with a suitable tool](./media/app-insights-export-telemetry/06-json.png)

Time durations are in ticks, where 10 000 ticks = 1ms. For example, these values show a time of 10ms to send a request from the browser, 30ms to receive it, and 1.8s to process the page in the browser:

	"sendRequest": {"value": 10000.0},
	"receiveRequest": {"value": 30000.0},
	"clientProcess": {"value": 17970000.0}



## Processing the data

On a small scale, you can write some code to pull apart your data, read it into a spreadsheet, and so on. For example:

    private IEnumerable<T> DeserializeMany<T>(string folderName)
    {
      var files = Directory.EnumerateFiles(folderName, "*.blob", SearchOption.AllDirectories);
      foreach (var file in files)
      {
         using (var fileReader = File.OpenText(file))
         {
            string fileContent = fileReader.ReadToEnd();
            IEnumerable<string> entities = fileContent.Split('\n').Where(s => !string.IsNullOrWhiteSpace(s));
            foreach (var entity in entities)
            {
                yield return JsonConvert.DeserializeObject<T>(entity);
            }
         }
      }
    }

For a larger code sample, see [using a worker role][exportasa].



## <a name="delete"></a>Delete your old data
Please note that you are responsible for managing your storage capacity and deleting the old data if necessary. 

## If you regenerate your storage key...

If you change the key to your storage, continuous export will stop working. You'll see a notification in your Azure account. 

Open the Continuous Export blade and edit your export. Edit the Export Destination, but just leave the same storage selected. Click OK to confirm.

![Edit the continuous export, open and close thee export destination.](./media/app-insights-export-telemetry/07-resetstore.png)

The continuous export will restart.

## Export to Power BI

[Microsoft Power BI](https://powerbi.microsoft.com/) presents your data in rich and varied visuals, with the ability to bring together information from multiple sources. You can stream telemetry data about the performance and usage of your apps from Application Insights to Power BI.

[Stream Application Insights to Power BI](app-insights-export-power-bi.md)

![Sample of Power BI view of Application Insights usage data](./media/app-insights-export-telemetry/210.png)

## Export to SQL

Another option is to move the data to a SQL database, where you can perform more powerful analytics.

We have samples showing two alternative methods of moving the data from the blob storage to a database:

* [Export to SQL using a worker role][exportcode]
* [Export to SQL using Stream Analytics][exportasa]


On larger scales, consider [HDInsight](http://azure.microsoft.com/services/hdinsight/) - Hadoop clusters in the cloud. HDInsight provides a variety of technologies for managing and analyzing big data.



## Q & A

* *But all I want is a one-time download of a chart.*  
 
    Yes, you can do that. At the top of the blade, click [Export Data](app-insights-metrics-explorer.md#export-to-excel).

* *I set up an export, but there's no data in my store.*

    Did Application Insights receive any telemetry from your app since you set up the export? You'll only receive new data.

* *I tried to set up an export, but was denied access*

    If the account is owned by your organization, you have to be a member of the owners or contributors groups.

    <!-- Your account has to be either a paid-for account, or in the free trial period. -->

* *Can I export straight to my own on-premises store?* 

    No, sorry. Our export engine currently only works with Azure storage at this time.  

* *Is there any limit to the amount of data you put in my store?* 

    No. We'll keep pushing data in until you delete the export. We'll stop if we hit the outer limits for blob storage, but that's pretty huge. It's up to you to control how much storage you use.  

* *How many blobs should I see in the storage?*

 * For every data type you selected to export, a new blob is created every minute (if data is available). 
 * In addition, for applications with high traffic, additional partition units are allocated. In this case each unit creates a blob every minute.


* *I regenerated the key to my storage or changed the name of the container, and now the export doesn't work.*

    Edit the export and open the export destination blade. Leave the same storage selected as before, and click OK to confirm. Export will restart. If the change was within the past few days, you won't lose data.

* *Can I pause the export?*

    Yes. Click Disable.


<!--Link references-->

[exportcode]: app-insights-code-sample-export-telemetry-sql-database.md
[exportasa]: app-insights-code-sample-export-sql-stream-analytics.md
[roles]: app-insights-resources-roles-access-control.md

 
