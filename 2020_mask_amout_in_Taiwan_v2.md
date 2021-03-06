## Linebot-GAS Query The amount of mask in Taiwan , Version 2

Fix:
1. The amount of mask is more to list first 
2. Replace all "臺" to "台" in address field
3. Search 全台(less 15 records)
4. new data source : https://data.nhi.gov.tw/Datasets/Download.ashx?rid=A21030000I-D50001-001&l=https://data.nhi.gov.tw/resource/mask/maskdata.csv (2020/02/11)
5. Add a lock mechanism to avoid  the problem during updating simultaneously (2020/02/12)


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
    var replyMessage="";
    var modified_timestamp=record_sheet.getRange(1, 2).getValue()
    var now_timestamp=new Date().getTime()

    if((now_timestamp-modified_timestamp)>180*1000){//Update if over 3 minutes(180*1000ms)
        var lock = LockService.getScriptLock();
        lock.waitLock(20000); // Lock 20 second,Ref:https://www.wfublog.com/2017/03/google-apps-script-spreadsheet-delay-write-data.html

        var response=UrlFetchApp.fetch(s_url);
        if(response != false){
          //Import csvData to Sheet, Ref:https://www.labnol.org/code/20279-import-csv-into-google-spreadsheet
          var csvData = Utilities.parseCsv(response.getContentText().replace(/臺/g, "台").replace(/０/g, "0").replace(/１/g, "1").replace(/２/g, "2").replace(/３/g, "3").replace(/４/g, "4").replace(/５/g, "5").replace(/６/g, "6").replace(/７/g, "7").replace(/８/g, "8").replace(/９/g, "9"), ",");      
          //for(var j=1;j<csvData.length;j++)
            //csvData[j][2]=csvData[j][2].replace(/臺/g, "台");//replace all 臺 to 台 in address field
          mask_sheet.clearContents();//Ref:https://developers.google.com/apps-script/reference/spreadsheet/sheet#clearContents()
          mask_sheet.clearContents();//Ref:https://developers.google.com/apps-script/reference/spreadsheet/sheet#clearContents()
          mask_sheet.getRange(1, 1, csvData.length, csvData[0].length).setValues(csvData);
          
          //無法排出我想要的順序:先對[縣市-鄕鎮區]排後,再對[成人口罩數量排序],要解決此問題,需將地址切成3個欄位,[縣市,鄉鎮區,剩餘地址]
          //mask_sheet.getRange(2, 1, mask_sheet.getLastRow(), mask_sheet.getLastColumn()).sort([{ column : 3,ascending: true },{column: 5,ascending: false }] );
          //對[成人口罩數量]倒排序:這樣資料列出時會先列剩餘口罩最多的
          mask_sheet.getRange(2, 1, mask_sheet.getLastRow(), mask_sheet.getLastColumn()).sort([{column: 5,ascending: false }] );
          //Logger.log(csvData.length)
          record_sheet.getRange(1, 2).setValue(new Date().getTime());
          lock.releaseLock(); 
        }
      }
      var mask_LastRow = mask_sheet.getLastRow();
      var county=userMessage.replace(/臺/g, "台")
      //var flag=0
    
      var count=0;
      data=mask_sheet.getRange(1, 1, mask_LastRow, 7).getValues();
      for(var i=1;i<mask_LastRow;i++){
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
         replyMessage="找不到任何資料!!!\n請輸入[縣市]或是[縣市,鄉鎮區]或全台\n如:\n高雄市\n高雄市新興區\n全台\n";
      }else{
         replyMessage+="查詢方式 [縣市]或是[縣市,鄉鎮區]或全台\n如:\n高雄市\n高雄市新興區\n全台\n";
      }
      record_sheet.getRange(2, 2).setValue(record_sheet.getRange(2, 2).getValue()+1);
      record_sheet.getRange(2, 3).setValue(new Date());//記錄最後存取時間
      record_sheet.getRange(2, 4).setValue(county);//記錄最後下的指令
      record_sheet.getRange(2, 5).setValue("");//記錄錯誤-設為空(不記錄)

  }catch (e) {
    replyMessage="發生錯誤!!";
    record_sheet.getRange(2, 5).setValue("發生錯誤!!"+e);//記錄錯誤 若發生"Exception: 範圍的座標在工作表的涵蓋面積之外",代表工作表不夠,要再新增row數
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

執行記錄:  
[20-02-08 17:15:46:049 HKT] SpreadsheetApp.Sheet.getRange([1, 2]) [0 秒]  
[20-02-08 17:15:46:140 HKT] SpreadsheetApp.Range.getValue() **[0.091 秒]**  
[20-02-08 17:15:46:149 HKT] console.log([73180.0, []]) [0.003 秒]  
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getLastRow() [0.083 秒]  
[20-02-08 17:15:46:233 HKT] SpreadsheetApp.Sheet.getRange([1, 1, 5510, 7]) [0 秒]  
[20-02-08 17:15:46:829 HKT] SpreadsheetApp.Range.getValues() [0.594 秒]  
...  
[20-02-08 17:15:47:245 HKT] 執行成功 [總執行時間：1.115 秒]  
