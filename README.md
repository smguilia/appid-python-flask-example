# IBM Bluemix App ID sample

This example is meant for developers who want to use the [IBM Bluemix App ID](https://console.ng.bluemix.net/docs/services/appid/index.html) service to protect their python server, or to create an end-to-end flow authentication with python.

## Specify your protected resource

1. Connect to a protected resource. A protected resource is an endpoint that requires authentication before a user can connect. In this example, the endpoint is called `/protected`.

  ```python
  from flask import Flask, session,render_template
  WebAppStrategy['AUTH_CONTEXT'] = "APPID_AUTH_CONTEXT";
  @app.route('/protected')
  def protected():
      tokens = session.get(WebAppStrategy['AUTH_CONTEXT'])
      if (tokens):
          publickey = retrievePublicKey(ServiceConfig.serverUrl)
          pem = getPublicKeyPem(publickey)
          idToken = tokens.get('id_token')
          accessToken = tokens.get('access_token')
          idTokenPayload = verifyToken(idToken,pem)
          accessTokenPayload =verifyToken(accessToken,pem)
          if (not idTokenPayload or not accessTokenPayload):
              session[WebAppStrategy['AUTH_CONTEXT']]=None
              return startAuthorization()
          else:
              print('idTokenPayload')
              print (idTokenPayload)
              return render_template('protected.html', name=idTokenPayload.get('name'),picture=idTokenPayload.get('picture'))
      else:
          return startAuthorization()
  ```

2. Verify that the user has a valid token. This exampe stores the token in `APPID_AUTH_CONTEXT`.

  * If a user has a valid token, they are able to access the protected resource. If the user is not authenticated, or has an invalid token, the authorization process must start.


## Authorization process

You can grant a user access to a protected resource by using the authorization process. At the end of the process, you can verify a user by reviewing their token.


1. Redirect the user to the app-id endpoint. The `client id` and `redirect URL` should be used as query parameters.

  ```python
  @app.route('/startAuthorization')
  def startAuthorization():
      serviceConfig=ServiceConfig()
      clientId = serviceConfig.clientId

      authorizationEndpoint = serviceConfig.serverUrl + AUTHORIZATION_PATH
      redirectUri = serviceConfig.redirectUri
      return redirect("{}?client_id={}&response_type=code&redirect_uri={}&scope=appid_default".format(authorizationEndpoint,clientId,redirectUri))
  ```

2. Bind the service to your app. Your application global environment contains App ID credentials. The credentials include your client id, server URL, and more. I've created a class called `serviceConfig` that reads the data. You can use this class or copy the data from the service credentials tab in your App ID dashboard. After the redirect occurs, the login widget is presented to the user.

3. After authorization, the user is redirected to your redirect endpoint with code that can be exchanged for an id token and access token.

  ```python
  @app.route('/afterauth')
  def afterauth():
      error = request.args.get('error')
      code = request.args.get('code')
      if error:
          return error
      elif code:
          return handleCallback(code)
      else:
          return '?'
  ```
  * If there is an error, such as the user didn't grant access to your app I return the code to the user. If you're using this in a production app, you could redirect them back to the login screen or allow them to continue unauthenticated to unprotected resources.

4. Replace the code with an access token and an identity token. Send a post request to the token endpoint using the following variables:

  <table>
  <tr>
    <th> Variable </th>
    <th> Definition </th>
  </tr>
  <tr>
    <td> <i> client id </i> </td>
    <td> This can be found in the service credentials tab of your service dashboard. If your app is bound to App ID, the serviceConfig class will extract it. </td>
  </tr>
  <tr>
    <td> <i> grunt_type </i> </td>
    <td> Will always be <i>authorization_code</i>. </td>
  </tr>
  <tr>
    <td> <i> redirect_uri </i> </td>
    <td> The redirect URI you used in step 1. The service validates the URI in the code exchange process. </td>
  </tr>
  <tr>
    <td> <i> code </i> </td>
    <td> The code your received after step 1. </td>
  </tr>
  </table>

  ```python
  def retriveTokens(grantCode):
      serviceConfig=ServiceConfig()
      clientId = serviceConfig.clientId
      secret = serviceConfig.secret
      tokenEndpoint = serviceConfig.serverUrl + TOKEN_PATH
      redirectUri = serviceConfig.redirectUri
  #    requests.post(url, data={}, auth=('user', 'pass'))
      r = requests.post(tokenEndpoint, data={"client_id": clientId,"grant_type": "authorization_code","redirect_uri": redirectUri,"code": grantCode
  		}, auth=HTTPBasicAuth(clientId, secret))
      print(r.status_code, r.reason)
      if (r.status_code is not 200):
          return 'fail'
      else:
          return r.json()

  def handleCallback(grantCode):
      tokens=retriveTokens(grantCode)
      if (type(tokens) is str):
          return tokens#it's error
      else:
          if (tokens['access_token']):
              session[WebAppStrategy['AUTH_CONTEXT']]=tokens
              return protected()
          else:
              return 'fail'
  ```
  * If no error is returned, a JSON file contains the two tokens. In my code, I saved them for the session and redirected to the protected resource for validation.

5. Validate the token. App ID tokens are signed with a private key. To validate, use the code in `token-utils.py` to open the key. I used the pyJWT library to handle jwt tokens.

  ```python
      import jwt
      PUBLIC_KEY_PATH = "/publickey";
      publickey = retrievePublicKey(ServiceConfig.serverUrl)
      pem = getPublicKeyPem(publickey)
      token = '{{some token}}'
      verifyToken(token,pem)
      def verifyToken(token,pemVal):
          try:
              payload = jwt.decode(token, pemVal, algorithms=['RS256'], options={'verify_aud':False})
              print('verified')
              return payload
       except:
              print ('not verified')
              return False
      def retrievePublicKey(serverUrl):
          serverUrl = serverUrl + PUBLIC_KEY_PATH;
          content = urllib2.urlopen(serverUrl).read()
          publicKeyJson=content;
          return  publicKeyJson
      def getPublicKeyPem(publicKeyJson=publicKeyJson):
          #some code I found in the internet to convert RS256 to pem
  ```


Congrats! Once you've finished the process, your website is fully protected.
