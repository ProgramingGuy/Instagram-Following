# Instagram, Non-Folllow Back
Generates a list of people not following back for Instagram.
Just paste code into DEV console in chrome while logged into your account on your profile page, it'll run and log an array of users.

```javascript
// Utility to get a cookie by name
function getCookie(name) {
  const cookies = `; ${document.cookie}`;
  const parts = cookies.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(';').shift();
}

// Utility to sleep for a given number of milliseconds
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Generate the next page URL for fetching followers
function generateNextPageUrl(after) {
  return `https://www.instagram.com/graphql/query/?query_hash=3dec7e2c57367ef3da3d987d89f9dbc8&variables=${encodeURIComponent(JSON.stringify({
    id: ds_user_id,
    include_reel: true,
    fetch_mutual: false,
    first: 24,
    after: after
  }))}`;
}

// Initialize variables
const csrftoken = getCookie("csrftoken");
const ds_user_id = getCookie("ds_user_id");

let initialURL = `https://www.instagram.com/graphql/query/?query_hash=3dec7e2c57367ef3da3d987d89f9dbc8&variables=${encodeURIComponent(JSON.stringify({
  id: ds_user_id,
  include_reel: true,
  fetch_mutual: false,
  first: 24
}))}`;

let totalFollowing = null;
let doNext = true;
let usersNotFollowingBack = [];
let fetchedCount = 0;
let sleepCycle = 0;

// Main script function
async function startScript() {
  while (doNext) {
    let responseJson;
    try {
      const response = await fetch(initialURL);
      responseJson = await response.json();
    } catch (error) {
      console.error("Fetch failed, retrying...", error);
      continue;
    }

    const userData = responseJson?.data?.user;
    if (!userData) {
      console.error("User data not found in response.");
      break;
    }

    if (!totalFollowing) {
      totalFollowing = userData.edge_follow.count;
    }

    const pageInfo = userData.edge_follow.page_info;
    doNext = pageInfo.has_next_page;
    initialURL = generateNextPageUrl(pageInfo.end_cursor);

    const edges = userData.edge_follow.edges;
    fetchedCount += edges.length;

    for (const { node } of edges) {
      if (!node.follows_viewer) {
        usersNotFollowingBack.push({ username: node.username, id: node.id });
      }
    }

    // Log progress
    console.clear();
    console.log(`Progress: ${fetchedCount}/${totalFollowing} (${Math.round(100 * fetchedCount / totalFollowing)}%)`);
    usersNotFollowingBack.forEach(user => {
      console.log(`Username: ${user.username}, ID: ${user.id}`);
    });

    // Sleep to avoid rate limits
    await sleep(Math.floor(400 * Math.random()) + 1000);
    sleepCycle++;

    if (sleepCycle > 6) {
      console.log("Sleeping to avoid instagram block...");
      await sleep(10000);
      sleepCycle = 0;
    }
  }

  // Save results as a JSON file
  const blob = new Blob([JSON.stringify(usersNotFollowingBack, null, 2)], { type: "application/json" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = "IG_NotFollowingBack.json";
  link.click();

  console.log("Script ran, data stored in array.");
}

startScript();
```
