## Linebot-GAS Query The amount of mask in Taiwan

1. Create a Line Bot on [LineDeveloper web ](https://developers.line.biz/zh-hant/)
2. Spreadsheet on [Google Drive](https://drive.google.com)
   This Spreadsheet has two Sheet, one's name is "口罩", another name is "記錄".
3. Create a Google Apps Script on [Google Drive](https://drive.google.com) & deploy it
```
  function doPost(e) {
    var CHANNEL_ACCESS_TOKEN = '*LineBot's channel access token*';
    var msg = JSON.parse(e.postData.contents);
    console.log(msg);
    // 取出 replayToken 和發送的訊息文字
    var replyToken = msg.events[0].replyToken;
    var userMessage = msg.events[0].message.text.trim();
    var userID = msg.events[0].source.userId;
    var replyMessage="";

    var s_url="http://data.nhi.gov.tw/Datasets/Download.ashx?rid=A21030000I-D50001-001&l=https://data.nhi.gov.tw/resource/mask/maskdata.csv";
    var Spreadsheet = SpreadsheetApp.openByUrl('The url of the Spreadsheet  on Google Drive'); //此處填入Google試算表的網址
    var mask_sheet = Spreadsheet.getSheetByName("口罩");
    var record_sheet = Spreadsheet.getSheetByName("記錄");

    var modified_timestamp=record_sheet.getRange(1, 2).getValue()
    var now_timestamp=new Date().getTime()
    console.log(now_timestamp-modified_timestamp);
    if((now_timestamp-modified_timestamp)>180000){//Update if over 3 minutes(180*1000ms)
      var response=UrlFetchApp.fetch(s_url);
      if(response != false){
        //Import csvData to Sheet, Ref:https://www.labnol.org/code/20279-import-csv-into-google-spreadsheet
        var csvData = Utilities.parseCsv(response.getContentText(), ",");    
        mask_sheet.clearContents();//Ref:https://developers.google.com/apps-script/reference/spreadsheet/sheet#clearContents()
        mask_sheet.getRange(1, 1, csvData.length, csvData[0].length).setValues(csvData);
          //Logger.log(csvData.length)
        record_sheet.getRange(1, 2).setValue(new Date().getTime())
        console.log("更新資料");
      }
    }
    var mask_LastRow = mask_sheet.getLastRow();
    var county=userMessage.replace("台", "臺")
    //var flag=0

    var count=0;
    data=mask_sheet.getRange(1, 1, mask_LastRow, 7).getValues();

    for(var i=1;i<mask_LastRow;i++){
      if(data[i][2].indexOf(county)>-1){
        replyMessage+=data[i][1]+"\n";
        replyMessage+="成人口罩量:"+data[i][4]+"\n兒童口罩量:"+data[i][5]+"\n";
        replyMessage+=data[i][2]+"\n"+data[i][3]+"\n更新時間:"+data[i][6]+"\n\n";
        count++;
        //flag=1;
      }else{
        //if(flag)
          //break;
      }
      if(count>15)
        break;
    }
    if(replyMessage===""){
       replyMessage="找不到任何資料!!!\n請輸入[縣市]或是[縣市,鄉鎮區]\n如:\n高雄市\n高雄市新興區";
    }else{
       replyMessage+="查詢方式 [縣市]或是[縣市,鄉鎮區]\n如:\n高雄市\n高雄市新興區\n";
    }
    record_sheet.getRange(2, 2).setValue(record_sheet.getRange(2, 2).getValue()+1);
    //console.log(replyMessage);


   var url = 'https://api.line.me/v2/bot/message/reply';
    UrlFetchApp.fetch(url, {
        'headers': {
        'Content-Type': 'application/json; charset=UTF-8',
        'Authorization': 'Bearer ' + CHANNEL_ACCESS_TOKEN,
      },
      'method': 'post',
      'payload': JSON.stringify({
        'replyToken': replyToken,
        'messages': [{
          'type': 'text',
          'text': replyMessage,
        }],
      }),
    });

  }//doPost(e)
```

tricky:
若使用getValue()來取得欄位值，執行速度會很慢  
getValue()執行一次可能就要耗掉0.091(如下記錄)  
下面記錄是執行取得10筆資料就要耗掉1.115 秒  
因此為了加速,上面程式碼則使用方法是將所有資料一次全部載入到記憶體(array),再執行判斷  
data=mask_sheet.getRange(1, 1, mask_LastRow, 7).getValues();  
資料大約5000筆,只需1秒以內就可以完成搜尋  


程式碼如下:
```
for(var i=2;i<10;i++){
      if(mask_sheet.getRange(i,3).getValue().indexOf(county)>-1){
        replyMessage+=mask_sheet.getRange(i,3).getValue()+"\n";
        ...
      }
```
```
執行記錄:
[20-02-08 17:15:46:049 HKT] SpreadsheetApp.Sheet.getRange([1, 2]) [0 秒]
[20-02-08 17:15:46:140 HKT] SpreadsheetApp.Range.getValue() [0.091 秒]
[20-02-08 17:15:46:149 HKT] console.log([73180.0, []]) [0.003 秒]
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getLastRow() [0.083 秒]
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getRange([1, 1, 5510, 7]) [0 秒]
[20-02-08 17:15:46:829 HKT] SpreadsheetApp.Range.getValues() [0.594 秒]
...
[20-02-08 17:15:47:245 HKT] 執行成功 [總執行時間：1.115 秒]
```
