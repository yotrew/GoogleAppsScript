## Linebot-GAS Query The amount of mask in Taiwan , Version 3

Fix:
1. The amount of mask is more to list first 
2. Replace all "臺" to "台" in address field
3. Search 全台(less 15 records)
4. new data source : https://data.nhi.gov.tw/Datasets/Download.ashx?rid=A21030000I-D50001-001&l=https://data.nhi.gov.tw/resource/mask/maskdata.csv (2020/02/11)
5. Add a lock mechanism to avoid  the problem during updating simultaneously (2020/02/12)
6. Avoid some data is lost.(2020/02/15)


1. Create a Line Bot on [LineDeveloper web ](https://developers.line.biz/zh-hant/)
2. Spreadsheet on [Google Drive](https://drive.google.com)
   This Spreadsheet has two Sheet, one's name is "口罩", another name is "記錄".
3. Create a Google Apps Script on [Google Drive](https://drive.google.com) & deploy it
```
function getDateTime(date_f,f){
    f = f || 1; 
	var timeDate= new Date(date_f);
	var tMonth = (timeDate.getMonth()+1) > 9 ? (timeDate.getMonth()+1) : '0'+(timeDate.getMonth()+1);
	var tDate = timeDate.getDate() > 9 ? timeDate.getDate() : '0'+timeDate.getDate();
	var tHours = timeDate.getHours() > 9 ? timeDate.getHours() : '0'+timeDate.getHours();
	var tMinutes = timeDate.getMinutes() > 9 ? timeDate.getMinutes() : '0'+timeDate.getMinutes();
	var tSeconds = timeDate.getSeconds() > 9 ? timeDate.getSeconds() : '0'+timeDate.getSeconds();
    if(f==1)
      return timeDate= timeDate.getFullYear()+'/'+ tMonth +'/'+ tDate +' '+ tHours +':'+ tMinutes +':'+ tSeconds;
    else if(f==2)
      return timeDate= timeDate.getFullYear()+'/'+ tMonth +'/'+ tDate;
    else if(f==3)
      return timeDate= tMonth +'/'+ tDate;
    else
      return timeDate= tHours +':'+ tMinutes +':'+ tSeconds;
}
  
function doPost(e) {
  var CHANNEL_ACCESS_TOKEN = '*LineBot's channel access token*';
  var msg = JSON.parse(e.postData.contents);
  console.log(msg);
  // 取出 replayToken 和發送的訊息文字
  var replyToken = msg.events[0].replyToken;
  var userMessage = msg.events[0].message.text.trim();
  var userID = msg.events[0].source.userId;
  var replyMessage="";
 
  try{
    var s_url="https://data.nhi.gov.tw/Datasets/Download.ashx?rid=A21030000I-D50001-001&l=https://data.nhi.gov.tw/resource/mask/maskdata.csv";
    var Spreadsheet = SpreadsheetApp.openByUrl('https://docs.google.com/spreadsheets/d/1fXZbbEtHt29f8KQDc1zzA0GxVWXeTfqwxk1mAGOkHtI/edit#gid=0'); //此處填入Google試算表的網址
    var mask_sheet = Spreadsheet.getSheetByName("口罩");
    var record_sheet = Spreadsheet.getSheetByName("記錄");
    var mask_LastRow = mask_sheet.getLastRow();
    
    var replyMessage="";
    
    var modified_timestamp=record_sheet.getRange(1, 2).getValue()
    var now_timestamp=new Date().getTime()
    //console.log("time interval:"+(now_timestamp-modified_timestamp));
    var data;

    if((now_timestamp-modified_timestamp)>180*1000){//Update if over 3 minutes(180*1000ms)
      var lock = LockService.getScriptLock();
      lock.waitLock(20000); // Lock 20 second,Ref:https://www.wfublog.com/2017/03/google-apps-script-spreadsheet-delay-write-data.html
      mask_sheet.getRange(2, 1, mask_sheet.getLastRow(), mask_sheet.getLastColumn()).sort([{column: 1,ascending: true }] );
      data=mask_sheet.getRange(1, 1, mask_LastRow, 7).getValues();
      var response=UrlFetchApp.fetch(s_url);
      if(response != false){
        //Import csvData to Sheet, Ref:https://www.labnol.org/code/20279-import-csv-into-google-spreadsheet
        var csvData = Utilities.parseCsv(response.getContentText().replace(/臺/g, "台").replace(/０/g, "0").replace(/１/g, "1").replace(/２/g, "2").replace(/３/g, "3").replace(/４/g, "4").replace(/５/g, "5").replace(/６/g, "6").replace(/７/g, "7").replace(/８/g, "8").replace(/９/g, "9"), ",");      
        console.log("csvData length:"+csvData.length+",maskdata length"+mask_LastRow);
        var ptr=0;
        for(var i=1;i<csvData.length;i++){
          for(var j=ptr;j<mask_LastRow;j++){
            if(csvData[i][0]==data[j][0]){
              //只更新成人口罩剩餘數和兒童口罩剩餘數
              data[j][4]=csvData[i][4];
              data[j][5]=csvData[i][5];
              data[j][6]=csvData[i][6];
              ptr=j+1;
              break;//break for j
            }
          }
          if(j>=mask_LastRow){//這是新資料
            //data.push(csvData[i]);
	    data[mask_LastRow]=csvData[i];
            //console.log("old data["+mask_LastRow+"]"+data[mask_LastRow-1]);
            mask_LastRow++;
            //console.log("new data["+mask_LastRow+"]"+data[mask_LastRow-1]);
            
          }
        }
        mask_sheet.clearContents();//Ref:https://developers.google.com/apps-script/reference/spreadsheet/sheet#clearContents()
        //mask_sheet.getRange(1, 1, csvData.length, csvData[0].length).setValues(csvData);
        mask_sheet.getRange(1, 1, data.length, data[0].length).setValues(data);
        
        //對[成人口罩數量]倒排序:這樣資料列出時會先列剩餘口罩最多的
        mask_sheet.getRange(2, 1, mask_sheet.getLastRow(), mask_sheet.getLastColumn()).sort([{column: 5,ascending: false }] );
        //Logger.log(csvData.length)
        record_sheet.getRange(1, 2).setValue(new Date().getTime());
        console.log("更新資料");
        lock.releaseLock(); 
      }
    }

    data=mask_sheet.getRange(1, 1, mask_LastRow, 7).getValues();
    var county=userMessage.replace(/臺/g, "台")
    //var flag=0;
    var count=0;

    for(var i=0;i<mask_LastRow;i++){
      if(data[i][2].indexOf(county)>-1 || county=="全台"){
        replyMessage+=data[i][1]+"\n";
        replyMessage+="成人口罩量:"+data[i][4]+"\n兒童口罩量:"+data[i][5]+"\n";
        replyMessage+=data[i][2]+"\n"+data[i][3]+"\n更新時間:"+getDateTime(data[i][6])+"\n\n";
        count++;
        //flag=1;
      }else{
        //if(flag)
          //break;
      }
      if(count>14)
        break;
    }
    if(replyMessage===""){
       replyMessage="找不到任何資料(請3分鐘後再查詢)!!!\n請輸入[縣市]或是[縣市,鄉鎮區]或全台\n如:\n高雄市\n高雄市新興區\n全台\n";
    }else{
       replyMessage+="查詢方式 [縣市]或是[縣市,鄉鎮區]或全台\n如:\n高雄市\n高雄市新興區\n全台\n";
    }
    //record_sheet.getRange(2, 2).setValue(record_sheet.getRange(2, 2).getValue()+1);
    record_sheet.getRange(2, 3).setValue(new Date());//記錄最後存取時間
    record_sheet.getRange(2, 4).setValue(county);//記錄最後下的指令
    record_sheet.getRange(2, 5).setValue("");//記錄最後下的指令
    console.log(replyMessage);
  }catch (e) {
    replyMessage="發生錯誤!!(請3分鐘後再查詢或連絡管理者)"+e;
    record_sheet.getRange(2, 5).setValue("發生錯誤!!"+e);
    //console.log(e);
    //logMyErrors(e); // 將例外傳至例外處理機制
  }
  
 
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

執行記錄:  
[20-02-08 17:15:46:049 HKT] SpreadsheetApp.Sheet.getRange([1, 2]) [0 秒]  
[20-02-08 17:15:46:140 HKT] SpreadsheetApp.Range.getValue() **[0.091 秒]**  
[20-02-08 17:15:46:149 HKT] console.log([73180.0, []]) [0.003 秒]  
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getLastRow() [0.083 秒]  
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getRange([1, 1, 5510, 7]) [0 秒]  
[20-02-08 17:15:46:829 HKT] SpreadsheetApp.Range.getValues() [0.594 秒]  
...  
[20-02-08 17:15:47:245 HKT] 執行成功 [總執行時間：1.115 秒]  
