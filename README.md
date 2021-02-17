Changes to GitHub API authentication
Since the course was published, GitHub has depreciated authentication via URL query parameters You can get an access token by following these instructions For this app we don't need to add any permissions so don't select any in the scopes. DO NOT SHARE ANY TOKENS THAT HAVE PERMISSIONS This would leave your account or repositories vulnerable, depending on permissions set.

It would also be worth adding your default.json config file to .gitignore If git has been previously tracking your default.json file then...

git rm --cached config/default.json
Then add your token to the config file and confirm that the file is untracked with git status before pushing to GitHub. GitHub does have your back here though. If you accidentally push code to a repository that contains a valid access token, GitHub will revoke that token. Thanks GitHub 🙏

You'll also need to change the options object in routes/api/profile.js where we make the request to the GitHub API to...

const options = {
  uri: encodeURI(
    `https://api.github.com/users/${req.params.username}/repos?per_page=5&sort=created:asc`
  ),
  method: 'GET',
  headers: {
    'user-agent': 'node.js',
    Authorization: `token ${config.get('githubToken')}`
  }
};
npm package request depreciated
As of 11th February 2020 request has been depreciated and is no longer maintained. We already use axios in the client so we can easily change the above fetching of a users GitHub repositories to use axios.

Install axios in the root of the project

npm i axios
We can then remove the client installation of axios.

cd client
npm uninstall axios
Client use of the axios module will be resolved in the root, so we can still use it in client.

Change the above GitHub API request to..

const uri = encodeURI(
  `https://api.github.com/users/${req.params.username}/repos?per_page=5&sort=created:asc`
);
const headers = {
  'user-agent': 'node.js',
  Authorization: `token ${config.get('githubToken')}`
};

const gitHubResponse = await axios.get(uri, { headers });
You can see the full change in routes/api/profile.js

uuid no longer has a default export
The npm package uuid no longer has a default export, so in our client/src/actions/alert.js we need to change the import and use of this package.

change

import uuid from 'uuid';
to

import { v4 as uuidv4 } from 'uuid';
And where we use it from

const id = uuid();
to

const id = uuidv4();
Addition of normalize-url package 🌎
Depending on what a user enters as their website or social links, we may not get a valid clickable url. For example a user may enter traversymedia.com or www.traversymedia.com which won't be a clickable valid url in the UI on the users profile page. To solve this we brought in normalize-url to well.. normalize the url.

Regardless of what the user enters it will ammend the url accordingly to make it valid (assuming the site exists). You can see the use here in routes/api/profile.js

Fix broken links in gravatar 🔗
There is an unresolved issue with the node-gravatar package, whereby the url is not valid. Fortunately we added normalize-url so we can use that to easily fix the issue. If you're not seeing Gravatar avatars showing in your app then most likely you need to implement this change. You can see the code use here in routes/api/users.js

Redux subscription to manage local storage 📥
The rules of redux say that our reducers should be pure and do just one thing.

If you're not familiar with the concept of pure functions, they must do the following..

Return the same output given the same input.
Have no side effects.
So our reducers are not the best place to manage local storage of our auth token. Ideally our action creators should also just dispatch actions, nothing else. So using these for additional side effects like setting authentication headers is not the best solution here.

Redux provides us with a store.subscribe listener that runs every time a state change occurs.

We can use this listener to watch our store and set our auth token in local storage and axios headers accordingly.

if there is a token - store it in local storage and set the headers.
if there is no token - token is null - remove it from storage and delete the headers.
The subscription can be seen in client/src/store.js

We also need to change our client/src/utils/setAuthToken.js so it now handles both the setting of the token in local storage and in axios headers.

With those two changes in place we can remove all setting of local storage from client/src/reducers/auth.js. And remove setting of the token in axios headers from client/src/actions/auth.js. This helps keep our code predictable, manageable and ultimately bug free.

Component reuse ♻️
The EditProfile and CreateProfile have been reduced to one component ProfileForm.js
The majority of this logic came from the refactrored EditProfile Component, which was initially changed to fix the issues with the use of useEffect we see in this component.

If you want to address the linter warnings in EditProfile then this is the component you are looking for.

Log user out on token expiration 🔐
If the Json Web Token expires then it should log the user out and end the authentication of their session.

We can do this using a axios interceptor together paired with creating an instance of axios.
The interceptor, well... intercepts any response and checks the response from our api for a 401 status in the response.
ie. the token has now expired and is no longer valid, or no valid token was sent.
If such a status exists then we log out the user and clear the profile from redux state.

You can see the implementation of the interceptor and axios instance in utils/api.js

Creating an instance of axios also cleans up our action creators in actions/auth.js, actions/profile.js and actions/post.js

Remove Moment 🗑️
As some of you may be aware, Moment.js which react-moment depends on has since become legacy code.
The maintainers of Moment.js now recommend finding an alternative to their package.

Moment.js is a legacy project, now in maintenance mode.
In most cases, you should choose a different library.
For more details and recommendations, please see Project Status in the docs.
Thank you.

Some of you in the course have been having problems installing both packages and meeting peer dependencies.
We can instead use the browsers built in Intl API.
First create a utils/formatDate.js file, with the following code...

function formatDate(date) {
  return new Intl.DateTimeFormat().format(new Date(date));
}

export default formatDate;
Then in our Education.js component, import the new function...

import formatDate from '../../utils/formatDate';
And use it instead of Moment...

<td>
  {formatDate(edu.from)} - {edu.to ? formatDate(edu.to) : 'Now'}
</td>
So wherever you use <Moment /> you can change to use the formatDate function.
Files to change would be...

Education.js
Experience.js
CommentItem.js
PostItem.js
ProfileEducation.js
ProfileExperience.js
If you're updating your project you will now be able to uninstall react-moment and moment as project dependencies.

