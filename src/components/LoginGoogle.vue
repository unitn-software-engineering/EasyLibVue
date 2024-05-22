<script setup>
    import { ref, onMounted } from 'vue'
    import { loggedUser, setLoggedUser, clearLoggedUser } from '../states/loggedUser.js'

    const HOST = import.meta.env.VITE_API_HOST || `http://localhost:8080`
    const API_URL = HOST+`/api/v1`

    const YOUR_CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
    const YOUR_REDIRECT_URI = import.meta.env.VITE_GOOGLE_REDIRECT_URI;

    // Parse query string to see if page request is coming from OAuth 2.0 server.
    var fragmentString = location.hash.substring(1);
    var params = {};
    var regex = /([^&=]+)=([^&]*)/g, m;
    while (m = regex.exec(fragmentString)) {
        params[decodeURIComponent(m[1])] = decodeURIComponent(m[2]);
    }
    if (Object.keys(params).length > 0 && params['state']) {
        if (params['state'] == localStorage.getItem('state')) {

            localStorage.setItem('oauth2-test-params', JSON.stringify(params) );

            myLogin( params.access_token );
        
        } else {
            console.log('State mismatch. Possible CSRF attack');
        }
    }

    function myLogin( googleToken ) {
        fetch(API_URL+'/authentications', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify( { googleToken: googleToken } ),
        })
        .then((resp) => resp.json()) // Transform the data into json
        .then(function(data) { // Here you get the data to modify as you please
            setLoggedUser(data)
            // loggedUser.token = data.token;
            // loggedUser.email = data.email;
            // loggedUser.id = data.id;
            // loggedUser.self = data.self;
            emit('login', loggedUser)
            return;
        })
        .catch( error => console.error(error) ); // If there is any error you will catch them here

    };

    // Function to generate a random state value
    function generateCryptoRandomState() {
        const randomValues = new Uint32Array(2);
        window.crypto.getRandomValues(randomValues);

        // Encode as UTF-8
        const utf8Encoder = new TextEncoder();
        const utf8Array = utf8Encoder.encode(
        String.fromCharCode.apply(null, randomValues)
        );

        // Base64 encode the UTF-8 data
        return btoa(String.fromCharCode.apply(null, utf8Array))
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=+$/, '');
    }

    // If there's an access token, try an API request.
    // Otherwise, start OAuth 2.0 flow.
    function trySampleRequest() {
        var params = JSON.parse(localStorage.getItem('oauth2-test-params'));
        if (params && params['access_token']) {
        var xhr = new XMLHttpRequest();
        xhr.open('GET',
            'https://www.googleapis.com/drive/v3/about?fields=user&' +
            'access_token=' + params['access_token']);
        xhr.onreadystatechange = function (e) {
            if (xhr.readyState === 4 && xhr.status === 200) {
            console.log(xhr.response);
            } else if (xhr.readyState === 4 && xhr.status === 401) {
            // Token invalid, so prompt for user permission.
            oauth2SignIn();
            }
        };
        xhr.send(null);
        } else {
        oauth2SignIn();
        }
    }

    onMounted( () => {
    });

    /*
    * Create form to request access token from Google's OAuth 2.0 server.
    */
    function oauth2SignIn() {

        // create random state value and store in local storage
        var state = generateCryptoRandomState();
        localStorage.setItem('state', state);

        // Google's OAuth 2.0 endpoint for requesting an access token
        var oauth2Endpoint = 'https://accounts.google.com/o/oauth2/v2/auth';

        // Create element to open OAuth 2.0 endpoint in new window.
        var form = document.createElement('form');
        form.setAttribute('method', 'GET'); // Send as a GET request.
        form.setAttribute('action', oauth2Endpoint);

        // Parameters to pass to OAuth 2.0 endpoint.
        var params = {'client_id': YOUR_CLIENT_ID,
                    'redirect_uri': YOUR_REDIRECT_URI,
                    'scope': 'https://www.googleapis.com/auth/userinfo.email',
                    'state': state,
                    'include_granted_scopes': 'true',
                    'response_type': 'token'};

        // Add form parameters as hidden input values.
        for (var p in params) {
            var input = document.createElement('input');
            input.setAttribute('type', 'hidden');
            input.setAttribute('name', p);
            input.setAttribute('value', params[p]);
            form.appendChild(input);
        }

        // Add form to page and submit it to open the OAuth 2.0 endpoint.
        document.body.appendChild(form);
        form.submit();
    };

</script>

<template>
    <button @click="oauth2SignIn();">oauth2SignIn</button>
</template>
