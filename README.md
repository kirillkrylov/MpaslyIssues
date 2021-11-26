## Issues

### Initial Sync makes 349 request[^5]
Most of the requests are unnecessary. Initial request should be limited to $metadata and minimum amount of fields to show default objects on the map.
![ScreenShot](Images/DefaultObjects.png).


### Excessive use of Login[^1]
During the initial synchronization mapsly makes **13 Login requests**, all of which return 200, and create 13 independent sessions. Sessions never disposed of (Logout never called).

### Excessive use $metadata[^2]
During the initial synchronization mapsly makes **13 Login requests**, all of which return the entire EDM model. These requests are heavy and take up to **9 SECONDS** to complete.

### Excessive Data transfer[^3]
Requests for the actual data, requests the entire record without the regard for user's preferences (fields to be syncronized). User's expectations are that only the _**necassy**_ fileds will be downloaded.


### Incorect OData json when creating items in mapsly
Mapsly offers an ability to create records by right clicking anywhere on the map, however every single one fails.
![ScreenShot1](Images/CreateContact.png)

The source of error is incorrect data transfered to creatio. 
```json
"CountryId": "Canada", // INCORRECT - CREATIO EXPECTS GUID OF THE COUNTRY NOT THE COUNTRY NAME
"RegionId": "Ontario", // INCORRECT - CREATIO EXPECTS GUID OF THE REGION NOT THE REGION NAME
"CityId": "Toronto",   // INCORRECT - CREATIO EXPECTS GUID OF THE CITY NOT THE CITY NAME
```
Mapsly's support recomendation to recompile Creatio will not solve this issue. See below for details

#### Request Sample when a new Contact is created
```http
POST /0/odata/$batch HTTP/1.1
Host: 4616-31-153-14-232.ngrok.io
User-Agent: GuzzleHttp/6.5.5 curl/7.74.0 PHP/7.4.21
Content-Length: 322
Accept: application/json
Bpmcsrf: 0XfLdmz6dMyryoMu6cG.PO
Content-Type: application/json;odata=verbose
Cookie: BPMLOADER=aw5lr4td1hwoqrfd1idq4gew; .ASPXAUTH=264F49E12D1F3B705755DBEB75C87AC237F6126413E32D04BE36E87F1ADCE6A05DC5DE8989F04B002A666BB471FA2D9901B2B59D80DA47CA80104CC40EE099A2A918D04800F6BFAF832A0C9DB5437B8A6240BAEDFA3CFFE77E74A6B3886159C2F46ECAA5F9E7ADB51E2DF263A8E5D535A2F61067B558313AB2F32F5F2F162505F091E794036308DCB43785BA44ADF725C001BD4A33C1F927CCBD7D757C18A47F565171DD2D93A73F8F0697AF6A0096B4721D8AF0A641D87D39AB6D8BE1250E6488FEFEB412CF9CD5566D6F4E233FBE65C05E6BAE81E4527B58E11742FBF6890C7010110CF9B38CE4668CD00B189E0898013D1F339531DFBD9691667284A41490B87985E61AA7A7C4597D42C7A5033F234FFB435F38BE0538F16ACD139E9465A6D5AF290E5070A769AC00626717057F13F4FE85612D6DA60B119AF68B207FC33CB8562F8AC7A194FE61767682D6161CE038C3EAA33A865B1512B146ED2FB2ACB62DAC5600B20F48A3100D8332C4227E3CC21133C56A4C22E29773D3F15C6DC437BC59FCD8719EB1935F7A552B6F91246614BFCAFAB4F4AB8022DDE4EC7FB24F1A213BF4C0706616DA8AB3CE98DB597D4550AD3A8507A4F27FDED29A1F597C3701672F6F68D859C507850C402E9B18FF05B3092D99DA003C7FE8463FB0A5438E1E00E94B12DABB6A62CCBF2DDDF2D405A0DFA7E1BC; BPMCSRF=0XfLdmz6dMyryoMu6cG.PO; UserName=83|117|112|101|114|118|105|115|111|114
Prefer: continue-on-error
X-Adapter-Request-Id: 07d4231a-a917-4acc-b8d2-722a554846e7
X-Forwarded-For: 107.21.255.92
X-Forwarded-Proto: http
Accept-Encoding: gzip
```

```json
{
	"requests": [
		{
			"method": "POST",
			"url": "Contact",
			"id": "0",
			"body": {
				"Zip": "M1T 2L6",
				"CountryId": "Canada",
				"RegionId": "Ontario",
				"CityId": "Toronto",
				"Address": "818 Huntingwood Dr",
				"Name": "TEST MAPSLY CONTACT"
			},
			"headers": {
				"Content-Type": "application/json;odata=verbose",
				"Accept": "application/json",
				"Prefer": "continue-on-error"
			}
		}
	]
}
```

##### RESPONSE
```json
{
	"responses": [
		{
			"id": "0",
			"status": 400,
			"headers": {
				"content-type": "application/json; odata.metadata=minimal",
				"odata-version": "4.0"
			},
			"body": {
				"error": {
					"code": "",
					"message": "The request is invalid.",
					"innererror": {
						"message": "item : Cannot convert the literal 'Canada' to the expected type 'Edm.Guid'.\r\n",
						"type": "",
						"stacktrace": ""
					}
				}
			}
		}
	]
}
```



### Use of global EntityEventListener
In **UsrAdapterNotifier.cs** schmea the following attrube is used `[EntityEventListener(IsGlobal = true)]` indicating that **ALL** database changes can be handled by this listener. 
- Handler sends Id of every record deleted from the system without anyr egard wheather or not such record is syncronized with Mapsly.
- Incorrect instantiation of HttpClient
	```cs
	private HttpClient getClient() {
		return new HttpClient();
	}
	```
	HttpClient is intended to be instantiated once and **re-used throughout the life of an application**. Instantiating an HttpClient class for every request **will exhaust** the number of sockets available under heavy loads. This will result in SocketException errors. Below is an example using HttpClient correctly. [See Official Microsoft Documentation](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=netframework-4.7.2)

## Footnotes

[^1]: |Verb|Path|Response|Duration|
      |:--|:--|:--|:--|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.14ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|45.4ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|43.15ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|45.33ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|66.94ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.86ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.9ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|50.38ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|62.96ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|116.05ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|117.18ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|125.32ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|93.18ms|


[^2]: |Verb|Path|Response|Duration|
      |:--|:--|:--|:--|
      |GET |/0/odata/$metadata|200 OK|2.86s|
      |GET |/0/odata/$metadata|200 OK|2.54s|
      |GET |/0/odata/$metadata|200 OK|2.56s|
      |GET |/0/odata/$metadata|200 OK|3.53s|
      |GET |/0/odata/$metadata|200 OK|3.28s|
      |GET |/0/odata/$metadata|200 OK|3.24s|
      |GET |/0/odata/$metadata|200 OK|4.86s|
      |GET |/0/odata/$metadata|200 OK|4.34s|
      |GET |/0/odata/$metadata|200 OK|7.53s|
      |GET |/0/odata/$metadata|200 OK|7.45s|
      |GET |/0/odata/$metadata|200 OK|6.98s|
      |GET |/0/odata/$metadata|200 OK|7.08s|
      |GET |/0/odata/$metadata|200 OK|7.13s|

[^3]: Contact Payload
      Excessive data is requested by maply. Instead of syncronizing contact's address, mapsly pulls the entire record.
     
      ```http
      GET /0/odata/Contact?$count=true&$top=100&$skip=0 HTTP/1.1
      Host: 4616-31-153-14-232.ngrok.io
      User-Agent: GuzzleHttp/6.5.5 curl/7.74.0 PHP/7.4.21
      Cookie: BPMLOADER=ejekcialtfa2gapiypq0xj5o; .ASPXAUTH=CC338E80EF9CC056880BA18C29B209582E8EB1618005CEAAF49E3D40B33AA74F33A93CADA338B9B9160983D5D1680FE7CE53539D9A8115F5825993881FA348F2CD8080622103DCD92A4E5645106B06D63908A5DC520EFD4B58D60DE3B2D377040D1C45BE462080CB021A5E93DF814BAE7263EE69ADC27D57348EB41DA0C9B5AFE1FCAEBA8C36264D1BB07DB16A85A290EFED7F4FDE0E92BB6E936AEFE6F666B52E5B90C15EAF0C9516ACB37A950EAF19EBAAF835FEA906111EBE9CFFB8C9F4A896E6A261E3CEE00DABF1885694353AFE3155E3B0A98AC3A114952CF02A3519FAB88CEA4954AFA1D8628384A65BCED8D9B47D01C49D23C2FFD9B505B1AAF33E0DB80291D86DB78CE700F8B72B454AF4FA2A178CC099633F999684D64336734DD318E1BE74CB0D036BE3E283ED8443F1C173E2A38F976C17D418BD789738EBDE2B19F4FEBD7F5934510D67574E83B5473F65DA0F5C9E77DA208FD0544768DD27B7DA9460D40BE1B742504046B8C304001802CF5E37A5BF6CF805D5731D2298C60B2603BC1B2130F56F7E5DC27EF93AED63FA09E53FD65A4A8DA04552ED3435A4E26FCE7D8C4673C37CABD661211FE0ED2E5629F680C6B286626F8082A9FEE95558D551D2CC9BFAF0A6E556626DF55A52A6A5295AF581B8953B1BB1E45A5EA6254FA292BAF59FD8A785E9C5E7D1E9376BB0E3E5B53B; BPMCSRF=zw7TrXbBzXGg0FVGOJHXGu; UserName=83|117|112|101|114|118|105|115|111|114; BPMSESSIONID=qsk5vwui5yy15bnsfzn55ukx
      X-Adapter-Request-Id: aab14a70-b75c-4f74-993f-25266b9dee0a
      X-Forwarded-For: 107.21.255.92
      X-Forwarded-Proto: http
      Accept-Encoding: gzip
      ```

      RESPONSE
      ```json
      {
      	"Id": "3b9f1667-165a-4dbf-b422-4d490c214605",
      	"Name": "Kirill Krylov, CPA",
      	"OwnerId": "410006e1-ca4e-4502-a9ec-e54d922d2c00",
      	"CreatedOn": "2021-11-10T15:42:37.32622Z",
      	"CreatedById": "410006e1-ca4e-4502-a9ec-e54d922d2c00",
      	"ModifiedOn": "2021-11-24T11:11:32.33577Z",
      	"ModifiedById": "410006e1-ca4e-4502-a9ec-e54d922d2c00",
      	"ProcessListeners": 0,
      	"Dear": "",
      	"SalutationTypeId": "00000000-0000-0000-0000-000000000000",
      	"GenderId": "00000000-0000-0000-0000-000000000000",
      	"AccountId": "00000000-0000-0000-0000-000000000000",
      	"DecisionRoleId": "00000000-0000-0000-0000-000000000000",
      	"TypeId": "00000000-0000-0000-0000-000000000000",
      	"JobId": "00000000-0000-0000-0000-000000000000",
      	"JobTitle": "",
      	"DepartmentId": "00000000-0000-0000-0000-000000000000",
      	"BirthDate": "0001-01-01T00:00:00Z",
      	"Phone": "",
      	"MobilePhone": "+380950661405",
      	"HomePhone": "",
      	"Skype": "",
      	"Email": "",
      	"AddressTypeId": "4f8b2d67-71d0-45fb-897e-cd4a308a97c0",
      	"Address": "1875 Broockshire Sq.",
      	"CityId": "e8749d0b-ea9f-4d1d-9fba-c182ac6af1a9",
      	"RegionId": "034e1fbd-7c28-4bdd-aed5-fdb720594e4c",
      	"Zip": "L1V 6L2",
      	"CountryId": "f8af0e88-f36b-1410-fa98-00155d043204",
      	"DoNotUseEmail": false,
      	"DoNotUseCall": false,
      	"DoNotUseFax": false,
      	"DoNotUseSms": false,
      	"DoNotUseMail": false,
      	"Notes": "",
      	"Facebook": "",
      	"LinkedIn": "",
      	"Twitter": "",
      	"FacebookId": "",
      	"LinkedInId": "",
      	"TwitterId": "",
      	"ContactPhoto@odata.mediaEditLink": "Contact(3b9f1667-165a-4dbf-b422-4d490c214605)/ContactPhoto",
      	"ContactPhoto@odata.mediaReadLink": "Contact(3b9f1667-165a-4dbf-b422-4d490c214605)/ContactPhoto",
      	"ContactPhoto@odata.mediaContentType": "application/octet-stream",
      	"TwitterAFDAId": "00000000-0000-0000-0000-000000000000",
      	"FacebookAFDAId": "00000000-0000-0000-0000-000000000000",
      	"LinkedInAFDAId": "00000000-0000-0000-0000-000000000000",
      	"PhotoId": "a1eb6c1a-2b9a-48be-8203-764959067342",
      	"GPSN": "",
      	"GPSE": "",
      	"Surname": "CPA",
      	"GivenName": "Kirill",
      	"MiddleName": "Krylov",
      	"Confirmed": true,
      	"IsNonActualEmail": false,
      	"Completeness": 30,
      	"LanguageId": "6ebc31fa-ee6c-48e9-81bf-8003ac03b019",
      	"Age": 0
      }
      ```



[^5]: |Verb|Path|Response|Duration|
      |:--|:--|:--|:--|
      |GET |/0/odata/Lead|200 OK|141.98ms|
      |GET |/0/odata/OpportunityType|200 OK|23.22ms|
      |GET |/0/odata/PartnerType|200 OK|21.9ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|22.21ms|
      |GET |/0/odata/LeadMedium|200 OK|23.38ms|
      |GET |/0/odata/LeadSource|200 OK|23.23ms|
      |GET |/0/odata/LeadRegisterMethod|200 OK|21.48ms|
      |GET |/0/odata/QualifyStatus|200 OK|23.31ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|21.67ms|
      |GET |/0/odata/Job|200 OK|22.45ms|
      |GET |/0/odata/Gender|200 OK|21.42ms|
      |GET |/0/odata/Department|200 OK|23.91ms|
      |GET |/0/odata/AccountCategory|200 OK|27.87ms|
      |GET |/0/odata/LeadDisqualifyReason|200 OK|25.57ms|
      |GET |/0/odata/LeadTypeStatus|200 OK|22.47ms|
      |GET |/0/odata/LeadType|200 OK|23.15ms|
      |GET |/0/odata/GeneratedWebForm|200 OK|23.37ms|
      |GET |/0/odata/Region|200 OK|24.19ms|
      |GET |/0/odata/Region|200 OK|37.62ms|
      |GET |/0/odata/Region|200 OK|40.38ms|
      |GET |/0/odata/Country|200 OK|36.32ms|
      |GET |/0/odata/Country|200 OK|38.94ms|
      |GET |/0/odata/AddressType|200 OK|23.2ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|22.77ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|21.71ms|
      |GET |/0/odata/AccountIndustry|200 OK|22.7ms|
      |GET |/0/odata/InformationSource|200 OK|22.4ms|
      |GET |/0/odata/LeadStatus|200 OK|23.76ms|
      |GET |/0/odata/ContactSalutationType|200 OK|37.02ms|
      |GET |/0/odata/Contact|200 OK|72.7ms|
      |GET |/0/odata/SysLanguage|200 OK|24.23ms|
      |GET |/0/odata/SysLanguage|200 OK|35.84ms|
      |GET |/0/odata/SysLanguage|200 OK|35.65ms|
      |GET |/0/odata/$metadata|200 OK|2.86s|
      |GET |/0/odata/SocialAccount|200 OK|28.15ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.14ms|
      |GET |/0/odata/Country|200 OK|42.78ms|
      |GET |/0/odata/Country|200 OK|39.36ms|
      |GET |/0/odata/OpportunityType|200 OK|20.99ms|
      |GET |/0/odata/Region|200 OK|23.55ms|
      |GET |/0/odata/PartnerType|200 OK|20.41ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|20.12ms|
      |GET |/0/odata/Region|200 OK|38.43ms|
      |GET |/0/odata/LeadMedium|200 OK|23.64ms|
      |GET |/0/odata/Region|200 OK|39.95ms|
      |GET |/0/odata/LeadSource|200 OK|34.96ms|
      |GET |/0/odata/AddressType|200 OK|40.01ms|
      |GET |/0/odata/LeadRegisterMethod|200 OK|22.9ms|
      |GET |/0/odata/Department|200 OK|25.52ms|
      |GET |/0/odata/QualifyStatus|200 OK|22.85ms|
      |GET |/0/odata/Job|200 OK|22.42ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|22.17ms|
      |GET |/0/odata/ContactType|200 OK|24.62ms|
      |GET |/0/odata/Job|200 OK|25.97ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|24.39ms|
      |GET |/0/odata/Gender|200 OK|22.87ms|
      |GET |/0/odata/Gender|200 OK|24.53ms|
      |GET |/0/odata/Department|200 OK|21.6ms|
      |GET |/0/odata/ContactSalutationType|200 OK|43.82ms|
      |GET |/0/odata/Account|200 OK|67.68ms|
      |GET |/0/odata/AccountCategory|200 OK|23.15ms|
      |GET |/0/odata/Pricelist|200 OK|22.66ms|
      |GET |/0/odata/LeadDisqualifyReason|200 OK|22.25ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|22.26ms|
      |GET |/0/odata/LeadTypeStatus|200 OK|22.79ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|24.42ms|
      |GET |/0/odata/LeadType|200 OK|22.84ms|
      |GET |/0/odata/AccountCategory|200 OK|22.88ms|
      |GET |/0/odata/GeneratedWebForm|200 OK|24.25ms|
      |GET |/0/odata/Country|200 OK|37.33ms|
      |GET |/0/odata/Region|200 OK|24.04ms|
      |GET |/0/odata/Region|200 OK|39.12ms|
      |GET |/0/odata/Country|200 OK|40.39ms|
      |GET |/0/odata/Region|200 OK|23.66ms|
      |GET |/0/odata/Region|200 OK|44.14ms|
      |GET |/0/odata/Region|200 OK|38.52ms|
      |GET |/0/odata/Country|200 OK|40.32ms|
      |GET |/0/odata/$metadata|200 OK|2.54s|
      |GET |/0/odata/Region|200 OK|45.82ms|
      |GET |/0/odata/Country|200 OK|42.74ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|45.4ms|
      |GET |/0/odata/AddressType|200 OK|32.42ms|
      |GET |/0/odata/AddressType|200 OK|23.45ms|
      |GET |/0/odata/Opportunity|200 OK|34.7ms|
      |GET |/0/odata/AccountType|200 OK|24.96ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|23.27ms|
      |GET |/0/odata/LeadType|200 OK|24.37ms|
      |GET |/0/odata/AccountIndustry|200 OK|39.09ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|23.27ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|23.15ms|
      |GET |/0/odata/SysLanguage|200 OK|22.64ms|
      |GET |/0/odata/AccountIndustry|200 OK|23.32ms|
      |GET |/0/odata/OpportunitySource|200 OK|26.66ms|
      |GET |/0/odata/SysLanguage|200 OK|35.85ms|
      |GET |/0/odata/InformationSource|200 OK|23.41ms|
      |GET |/0/odata/OpportunityMood|200 OK|31.58ms|
      |GET |/0/odata/SysLanguage|200 OK|34.62ms|
      |GET |/0/odata/LeadStatus|200 OK|23.76ms|
      |GET |/0/odata/OpportunityCategory|200 OK|21.95ms|
      |GET |/0/odata/SocialAccount|200 OK|30.66ms|
      |GET |/0/odata/ContactSalutationType|200 OK|21.69ms|
      |GET |/0/odata/OpportunityCloseReason|200 OK|26.5ms|
      |GET |/0/odata/Country|200 OK|41.73ms|
      |GET |/0/odata/OpportunityStage|200 OK|27.36ms|
      |GET |/0/odata/OpportunityType|200 OK|41.94ms|
      |GET |/0/odata/Country|200 OK|62.04ms|
      |GET |/0/odata/Region|200 OK|25.17ms|
      |GET |/0/odata/Region|200 OK|38.8ms|
      |GET |/0/odata/$metadata|200 OK|2.56s|
      |GET |/0/odata/Region|200 OK|39.81ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|43.15ms|
      |GET |/0/odata/AddressType|200 OK|22.33ms|
      |GET |/0/odata/Pricelist|200 OK|22.2ms|
      |GET |/0/odata/Department|200 OK|21.68ms|
      |GET |/0/odata/Job|200 OK|20.35ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|23.48ms|
      |GET |/0/odata/$metadata|200 OK|3.53s|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|21.62ms|
      |GET |/0/odata/ContactType|200 OK|25.43ms|
      |GET |/0/odata/AccountCategory|200 OK|23.24ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|23.73ms|
      |GET |/0/odata/Lead|200 OK|317.91ms|
      |GET |/0/odata/Country|200 OK|44.96ms|
      |GET |/0/odata/Gender|200 OK|21.79ms|
      |GET |/0/odata/$metadata|200 OK|3.28s|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|45.33ms|
      |GET |/0/odata/ContactSalutationType|200 OK|22.56ms|
      |GET |/0/odata/Country|200 OK|40.53ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|66.94ms|
      |GET |/0/odata/Region|200 OK|24.41ms|
      |GET |/0/odata/OpportunityType|200 OK|24.87ms|
      |GET |/0/odata/Region|200 OK|36.92ms|
      |GET |/0/odata/PartnerType|200 OK|23ms|
      |GET |/0/odata/LeadType|200 OK|47.09ms|
      |GET |/0/odata/Region|200 OK|41.74ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|22.87ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|27.26ms|
      |GET |/0/odata/AddressType|200 OK|22.86ms|
      |GET |/0/odata/LeadMedium|200 OK|23.27ms|
      |GET |/0/odata/OpportunitySource|200 OK|24.78ms|
      |GET |/0/odata/AccountType|200 OK|21.28ms|
      |GET |/0/odata/LeadSource|200 OK|26.83ms|
      |GET |/0/odata/OpportunityMood|200 OK|23.7ms|
      |GET |/0/odata/AccountIndustry|200 OK|21.2ms|
      |GET |/0/odata/LeadRegisterMethod|200 OK|25.76ms|
      |GET |/0/odata/OpportunityCategory|200 OK|25.63ms|
      |GET |/0/odata/QualifyStatus|200 OK|23.86ms|
      |GET |/0/odata/OpportunityCloseReason|200 OK|22.76ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|21.69ms|
      |GET |/0/odata/OpportunityStage|200 OK|23.68ms|
      |GET |/0/odata/Job|200 OK|24.36ms|
      |GET |/0/odata/OpportunityType|200 OK|25.93ms|
      |GET |/0/odata/$metadata|200 OK|3.24s|
      |GET |/0/odata/Gender|200 OK|22.36ms|
      |GET |/0/odata/Contact|200 OK|125.31ms|
      |GET |/0/odata/Department|200 OK|23.23ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.86ms|
      |GET |/0/odata/AccountCategory|200 OK|22.02ms|
      |GET |/0/odata/SysLanguage|200 OK|23.72ms|
      |GET |/0/odata/LeadDisqualifyReason|200 OK|22.91ms|
      |GET |/0/odata/SysLanguage|200 OK|34.83ms|
      |GET |/0/odata/LeadTypeStatus|200 OK|23.98ms|
      |GET |/0/odata/SysLanguage|200 OK|36.81ms|
      |GET |/0/odata/$metadata|200 OK|4.86s|
      |GET |/0/odata/LeadType|200 OK|25.81ms|
      |GET |/0/odata/Account|200 OK|112.91ms|
      |GET |/0/odata/SocialAccount|200 OK|25.9ms|
      |GET |/0/odata/GeneratedWebForm|200 OK|26.82ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|44.9ms|
      |GET |/0/odata/Region|200 OK|26.4ms|
      |GET |/0/odata/Country|200 OK|40.77ms|
      |GET |/0/odata/Pricelist|200 OK|27.77ms|
      |GET |/0/odata/Region|200 OK|45.5ms|
      |GET |/0/odata/$metadata|200 OK|4.34s|
      |GET |/0/odata/Country|200 OK|40.52ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|23.75ms|
      |GET |/0/odata/Region|200 OK|38.84ms|
      |GET |/0/odata/Region|200 OK|36.43ms|
      |GET |/0/odata/Opportunity|200 OK|107.71ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|21.09ms|
      |GET |/0/odata/Country|200 OK|39.2ms|
      |GET |/0/odata/Region|200 OK|40.77ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|50.38ms|
      |GET |/0/odata/AccountCategory|200 OK|23.17ms|
      |GET |/0/odata/Country|200 OK|48.77ms|
      |GET |/0/odata/Region|200 OK|40.14ms|
      |GET |/0/odata/Country|200 OK|53.75ms|
      |GET |/0/odata/AddressType|200 OK|23.28ms|
      |GET |/0/odata/LeadType|200 OK|25.48ms|
      |GET |/0/odata/AddressType|200 OK|22.15ms|
      |GET |/0/odata/Country|200 OK|42.67ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|26.98ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|25.52ms|
      |GET |/0/odata/Department|200 OK|23.39ms|
      |GET |/0/odata/Region|200 OK|23.31ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|23.35ms|
      |GET |/0/odata/OpportunitySource|200 OK|23.25ms|
      |GET |/0/odata/Job|200 OK|22.09ms|
      |GET |/0/odata/Region|200 OK|40.7ms|
      |GET |/0/odata/AccountIndustry|200 OK|25.46ms|
      |GET |/0/odata/OpportunityMood|200 OK|23.25ms|
      |GET |/0/odata/ContactType|200 OK|23.33ms|
      |GET |/0/odata/InformationSource|200 OK|23.59ms|
      |GET |/0/odata/OpportunityCategory|200 OK|24.09ms|
      |GET |/0/odata/Region|200 OK|41.69ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|28.15ms|
      |GET |/0/odata/LeadStatus|200 OK|26.51ms|
      |GET |/0/odata/OpportunityCloseReason|200 OK|25.17ms|
      |GET |/0/odata/Gender|200 OK|26.61ms|
      |GET |/0/odata/AddressType|200 OK|29.16ms|
      |GET |/0/odata/OpportunityStage|200 OK|30.46ms|
      |GET |/0/odata/ContactSalutationType|200 OK|49.08ms|
      |GET |/0/odata/AccountType|200 OK|28.34ms|
      |GET |/0/odata/ContactSalutationType|200 OK|47.02ms|
      |GET |/0/odata/OpportunityType|200 OK|40.08ms|
      |GET |/0/odata/AccountIndustry|200 OK|41.29ms|
      |GET |/0/odata/$metadata|200 OK|7.53s|
      |GET |/0/odata/$metadata|200 OK|7.45s|
      |GET |/0/odata/$metadata|200 OK|6.98s|
      |GET |/0/odata/$metadata|200 OK|7.08s|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|62.96ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|116.05ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|117.18ms|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|125.32ms|
      |GET |/0/odata/TradeMark|200 OK|35.33ms|
      |GET |/0/odata/ProductCategory|200 OK|32ms|
      |GET |/0/odata/ProductSource|200 OK|34.76ms|
      |GET |/0/odata/ProductType|200 OK|50.59ms|
      |GET |/0/odata/Tax|200 OK|48.81ms|
      |GET |/0/odata/Unit|200 OK|39.41ms|
      |GET |/0/odata/PartnerLevel|200 OK|49.67ms|
      |GET |/0/odata/ReasonForLeaving|200 OK|39.44ms|
      |GET |/0/odata/EmployeeJob|200 OK|46.96ms|
      |GET |/0/odata/OrgStructureUnit|200 OK|62.77ms|
      |GET |/0/odata/ChatQueue|200 OK|62.5ms|
      |GET |/0/odata/OmnichannelChatStatus|200 OK|39.69ms|
      |GET |/0/odata/DocumentState|200 OK|40.49ms|
      |GET |/0/odata/DocumentType|200 OK|38.24ms|
      |GET |/0/odata/ProblemStatus|200 OK|36.32ms|
      |GET |/0/odata/ProblemPriority|200 OK|35.6ms|
      |GET |/0/odata/ProblemType|200 OK|38.33ms|
      |GET |/0/odata/ReleasePriority|200 OK|45.28ms|
      |GET |/0/odata/ReleaseType|200 OK|39.85ms|
      |GET |/0/odata/ReleaseStatus|200 OK|43.36ms|
      |GET |/0/odata/ChangeStatus|200 OK|38.24ms|
      |GET |/0/odata/ChangePriority|200 OK|38.02ms|
      |GET |/0/odata/ChangeCategory|200 OK|40.35ms|
      |GET |/0/odata/ChangePurpose|200 OK|37.51ms|
      |GET |/0/odata/ChangeSource|200 OK|40.74ms|
      |GET |/0/odata/ConfigItemCategory|200 OK|31.43ms|
      |GET |/0/odata/ConfigItemStatus|200 OK|35.92ms|
      |GET |/0/odata/ConfigItemModel|200 OK|35.33ms|
      |GET |/0/odata/ConfigItemType|200 OK|51.72ms|
      |GET |/0/odata/ServiceStatus|200 OK|38.53ms|
      |GET |/0/odata/SupportLevel|200 OK|40.39ms|
      |GET |/0/odata/ServicePactType|200 OK|35.52ms|
      |GET |/0/odata/ServicePactStatus|200 OK|39.23ms|
      |GET |/0/odata/TimeUnit|200 OK|34.33ms|
      |GET |/0/odata/ServiceCategory|200 OK|37.47ms|
      |GET |/0/odata/RoleInServiceTeam|200 OK|45.41ms|
      |GET |/0/odata/CaseCategory|200 OK|43.3ms|
      |GET |/0/odata/SatisfactionLevel|200 OK|58.52ms|
      |GET |/0/odata/ClosureCode|200 OK|34.85ms|
      |GET |/0/odata/CaseOrigin|200 OK|40.88ms|
      |GET |/0/odata/CasePriority|200 OK|69.31ms|
      |GET |/0/odata/CaseStatus|200 OK|53.13ms|
      |GET |/0/odata/Territory|200 OK|42.73ms|
      |GET |/0/odata/EventStatus|200 OK|38.27ms|
      |GET |/0/odata/EventType|200 OK|39.25ms|
      |GET |/0/odata/BulkEmailAudienceSchema|200 OK|88.33ms|
      |GET |/0/odata/BulkEmailLaunchOption|200 OK|40.24ms|
      |GET |/0/odata/BulkEmailCategory|200 OK|35.68ms|
      |GET |/0/odata/BulkEmailSplit|200 OK|47.45ms|
      |GET |/0/odata/BulkEmailStatus|200 OK|36.6ms|
      |GET |/0/odata/BulkEmailType|200 OK|36.87ms|
      |GET |/0/odata/CampaignScheduledMode|200 OK|33.96ms|
      |GET |/0/odata/CampaignType|200 OK|41.52ms|
      |GET |/0/odata/SegmentStatus|200 OK|40.81ms|
      |GET |/0/odata/CampaignStatus|200 OK|43.1ms|
      |GET |/0/odata/PartnerType|200 OK|37.89ms|
      |GET |/0/odata/LeadMedium|200 OK|32.54ms|
      |GET |/0/odata/LeadSource|200 OK|47.46ms|
      |GET |/0/odata/LeadRegisterMethod|200 OK|43.72ms|
      |GET |/0/odata/QualifyStatus|200 OK|42.88ms|
      |GET |/0/odata/LeadDisqualifyReason|200 OK|39.18ms|
      |GET |/0/odata/LeadTypeStatus|200 OK|36.96ms|
      |GET |/0/odata/GeneratedWebForm|200 OK|60.2ms|
      |GET |/0/odata/InformationSource|200 OK|48.08ms|
      |GET |/0/odata/LeadStatus|200 OK|40.55ms|
      |GET |/0/odata/ContractState|200 OK|41.21ms|
      |GET |/0/odata/ContractType|200 OK|37.5ms|
      |GET |/0/odata/PaymentType|200 OK|43.51ms|
      |GET |/0/odata/DeliveryType|200 OK|41.5ms|
      |GET |/0/odata/SourceOrder|200 OK|57.18ms|
      |GET |/0/odata/OrderDeliveryStatus|200 OK|36.3ms|
      |GET |/0/odata/OrderPaymentStatus|200 OK|37.37ms|
      |GET |/0/odata/OrderStatus|200 OK|39.43ms|
      |GET |/0/odata/LeadType|200 OK|36.19ms|
      |GET |/0/odata/OpportunityDepartment|200 OK|41.1ms|
      |GET |/0/odata/OpportunitySource|200 OK|39.06ms|
      |GET |/0/odata/OpportunityMood|200 OK|39.4ms|
      |GET |/0/odata/OpportunityCategory|200 OK|53.72ms|
      |GET |/0/odata/OpportunityCloseReason|200 OK|41.05ms|
      |GET |/0/odata/OpportunityStage|200 OK|41.53ms|
      |GET |/0/odata/OpportunityType|200 OK|37.49ms|
      |GET |/0/odata/ProjectStatus|200 OK|38.45ms|
      |GET |/0/odata/ProjectType|200 OK|55.87ms|
      |GET |/0/odata/ProjectEntryType|200 OK|41.84ms|
      |GET |/0/odata/Currency|200 OK|46.52ms|
      |GET |/0/odata/AccountBillingInfo|200 OK|50.73ms|
      |GET |/0/odata/InvoicePaymentStatus|200 OK|42.76ms|
      |GET |/0/odata/CallDirection|200 OK|43.5ms|
      |GET |/0/odata/EmailType|200 OK|44.88ms|
      |GET |/0/odata/EmailSendStatus|200 OK|46.51ms|
      |GET |/0/odata/ActivityResult|200 OK|58.12ms|
      |GET |/0/odata/ActivityStatus|200 OK|42.53ms|
      |GET |/0/odata/ActivityCategory|200 OK|39.32ms|
      |GET |/0/odata/ActivityType|200 OK|39.33ms|
      |GET |/0/odata/ActivityPriority|200 OK|36.14ms|
      |GET |/0/odata/TimeZone|200 OK|28.21ms|
      |GET |/0/odata/TimeZone|200 OK|50.41ms|
      |GET |/0/odata/SysLanguage|200 OK|25.06ms|
      |GET |/0/odata/SysLanguage|200 OK|41.86ms|
      |GET |/0/odata/SysLanguage|200 OK|45.53ms|
      |GET |/0/odata/SocialAccount|200 OK|38.93ms|
      |GET |/0/odata/Department|200 OK|43.57ms|
      |GET |/0/odata/Job|200 OK|42.93ms|
      |GET |/0/odata/ContactType|200 OK|37.67ms|
      |GET |/0/odata/ContactDecisionRole|200 OK|40ms|
      |GET |/0/odata/Gender|200 OK|48.03ms|
      |GET |/0/odata/ContactSalutationType|200 OK|47.2ms|
      |GET |/0/odata/Pricelist|200 OK|46.18ms|
      |GET |/0/odata/AccountAnnualRevenue|200 OK|46.4ms|
      |GET |/0/odata/AccountEmployeesNumber|200 OK|46.54ms|
      |GET |/0/odata/AccountCategory|200 OK|43.79ms|
      |GET |/0/odata/Country|200 OK|42.12ms|
      |GET |/0/odata/Country|200 OK|60.55ms|
      |GET |/0/odata/Region|200 OK|30.02ms|
      |GET |/0/odata/Region|200 OK|43.93ms|
      |GET |/0/odata/Region|200 OK|85.33ms|
      |GET |/0/odata/AddressType|200 OK|42.15ms|
      |GET |/0/odata/AccountType|200 OK|50.16ms|
      |GET |/0/odata/AccountIndustry|200 OK|420.95ms|
      |GET |/0/odata/$metadata|200 OK|7.13s|
      |POST|/ServiceModel/AuthService.svc/Login|200 OK|93.18ms|
      
