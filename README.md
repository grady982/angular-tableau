# Repo Info
This is an example repo to show how to embed Tableau reports in Angular.

--

# Tableau Embedded in Angular

在 Angular 中嵌入 Tableau Report 需要解決以下幾點:

1. Tableau 嵌入代碼
2. Tableau 嵌入身分驗證(Connected Apps) 
3. 產出 Tableau 信任的 Token(JWT)

# Tableau 嵌入代碼

- 官方文件: **Embed Views into Webpages**
    
    [Embed Views into Webpages](https://help.tableau.com/current/pro/desktop/en-us/embed.htm)
    

首先照著官方文件步驟取得嵌入代碼，示意如下:

```html
<script type='module' src='http://my_tableau_server/javascripts/api/tableau.embedding.3.latest.min.js'></script>
<tableau-viz 
	id='tableau-viz' 
	src='http://my_tableau_server/t/DTD000/views/_0/1/86997236-9f27-4ab8-bab5-18834600a41f/d357784e-2c72-438f-85c2-266851fecfbd' 
	width='1000' 
	height='827' 
	hide-tabs 
	toolbar='bottom' >
</tableau-viz>
```

~~以為拿到這段 code 事情就結束了，原來才剛開始~~

首先我們把這段 code 放到對應的位置去：

- `script` 放 **index.html**
- `tableau-viz` 放 **app.component.ts**

**index.html**

```jsx
<!doctype html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>My Angular App</title>
        <base href="/">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="icon" type="image/x-icon" href="favicon.ico">
				<script type='module' src='http://my_tableau_server/javascripts/api/tableau.embedding.3.latest.min.js'></script>
    </head>
    <body>
        <app-root></app-root>
    </body>
</html>
```

app**.component.html**

```html
<tableau-viz 
	id='tableau-viz' 
	src='http://my_tableau_server/t/DTD000/views/_0/1/86997236-9f27-4ab8-bab5-18834600a41f/d357784e-2c72-438f-85c2-266851fecfbd' 
	width='1000' 
	height='827' 
	hide-tabs 
	toolbar='bottom' >
</tableau-viz>
```

> `tableau-viz` 是 Tableau 提供的 component，雖然我們已經透過 `script` 下載對應的資源檔，但很可惜這段 code 無法直接在 Angular 使用，因為 Angular 並不認識 `tableau-viz`
> 

解法： `schemas: [ CUSTOM_ELEMENTS_SCHEMA ]` 告訴 Angular module 說我要使用自定義的 element

```tsx
@NgModule({
  declarations: [
    TableauViewComponent
  ],
  imports: [
    CommonModule,
    TableauManagementRoutingModule
  ],
  schemas: [ CUSTOM_ELEMENTS_SCHEMA ]
})
export class AppModule { }
```

沒意外的話你會看到 Tableau Server 的登入畫面，因為 embedded 的 URL 就代表向 Tableau Server 發送請求取得 embedded content 因為沒有提供適當的 token 所以被會 redirect 到此畫面， 需要輸入帳號密碼的方式才能看到嵌入內容並不是我們想要的，所以接下來談談 **Connected Apps** Tableau Authentication 的機制

![renditionDownload](https://gist.github.com/assets/51692045/d4fae83a-86aa-4bc4-9f0f-11aa5639baf2)

實際上我們的 embedded 的 URL 會發送 request 

# Tableau 嵌入身分驗證(Connected Apps)

- 官方文件: **Configure Tableau Connected Apps to Enable SSO for Embedded Content**
    
    [Configure Tableau Connected Apps to Enable SSO for Embedded Content](https://help.tableau.com/current/server/en-us/connected_apps.htm)
    

Connected Apps 是官方提供的認證方式，詳細內容請看文件

![connectedapp_how](https://gist.github.com/assets/51692045/e4ed6cc7-dfea-4d29-94df-3b279b16d1f6)

**TLDR；**

設定 Tableau Server 信任我們的 APP 並取得 ClientID、SecretID、SecretValue，我們會透過這一組訊息在我們的 Webserver 組 JWT 提供給 Webpage 在請求 Tableau Embedded Content 的時候驗證通過，這樣一來就不會再被 redirect 到 Tableau 的登入畫面了。~~聽起來很簡單八~~

# 產出 Tableau 信任的 Token(JWT)

要如何產生 JWT ? 官方已經提供了 Java 和 Python 範例([**請看**](https://help.tableau.com/current/server/en-us/connected_apps.htm#step-3-configure-the-jwt))，那這邊在多提供一個 C# 範例請看 :

```csharp
using System.IdentityModel.Tokens.Jwt;
using Microsoft.IdentityModel.Tokens;
using System.Security.Claims;

public async Task<GenericRes> GenTableauToken()
{
				string secretId = _config["Tableau:SecretID"];
        string secretValue = _config["Tableau:SecretValue"];
        string clientId = _config["Tableau:ClientID"];
        string username = _config["Tableau:UserName"];
        string aud = _config["Tableau:Aud"];
        double tokenExpiryInMinutes = 10; // Max of 10 minutes.

        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(secretValue);

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(new[] {
                new Claim("sub",username)
                ,new Claim("aud",aud)
                ,new Claim("jti",DateTime.UtcNow.ToString("MM/dd/yyyy hh:mm:ss.fff tt"))
                ,new Claim("iss",clientId)
                ,new Claim("scp","tableau:views:embed")
            }),
            Expires = DateTime.UtcNow.AddMinutes(tokenExpiryInMinutes),
            SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
        };

        var token = tokenHandler.CreateJwtSecurityToken(tokenDescriptor);

        //client id
        token.Header.Add("iss", clientId);

        //secret id
        token.Header.Add("kid", secretId);

        return tokenHandler.WriteToken(token);
}
```

取得 JWT  token 後接著就是 binding 到 `tableau-viz` component 上 `[token]='my-JWT'`

**app.component.ts**

```html
<tableau-viz 
	id='tableau-viz' 
	src='http://my_tableau_server/t/DTD000/views/_0/1/86997236-9f27-4ab8-bab5-18834600a41f/d357784e-2c72-438f-85c2-266851fecfbd' 
	width='1000' 
	height='827' 
	hide-tabs 
	toolbar='bottom' 
	[token]='my-JWT'>
</tableau-viz>
```

**大功告成!! 🎉🎉🎉**

其他學習資源

[Tableau Embedded Analytics Playground - The Information Lab](https://playground.theinformationlab.io/)
