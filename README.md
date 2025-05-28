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

# Logs everyone and removes them automaticlly.
This will take a long time due to Instagram throttling, may be more efficent to manually remove everyone via the list.

```javascript
// === Helper Functions ===
function getCookie(name) {
  const cookies = `; ${document.cookie}`;
  const parts = cookies.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(";").shift();
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function generateNextPageUrl(after) {
  return `https://www.instagram.com/graphql/query/?query_hash=3dec7e2c57367ef3da3d987d89f9dbc8&variables=${encodeURIComponent(JSON.stringify({
    id: ds_user_id,
    include_reel: true,
    fetch_mutual: false,
    first: 24,
    after: after
  }))}`;
}

function unfollowUserUrl(id) {
  return `https://www.instagram.com/web/friendships/${id}/unfollow/`;
}

// === Main Script ===
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

// === Phase 1: Collect users who don't follow back ===
async function collectNonFollowers() {
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

    if (!totalFollowing) totalFollowing = userData.edge_follow.count;

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

    console.clear();
    console.log(`Progress: ${fetchedCount}/${totalFollowing} (${Math.round(100 * fetchedCount / totalFollowing)}%)`);
    usersNotFollowingBack.forEach(user => {
      console.log(`Username: ${user.username}, ID: ${user.id}`);
    });

    await sleep(Math.floor(400 * Math.random()) + 1000);
    sleepCycle++;
    if (sleepCycle > 6) {
      console.log("Sleeping 10 seconds to avoid temp block...");
      await sleep(10000);
      sleepCycle = 0;
    }
  }

  console.log(`\n Found ${usersNotFollowingBack.length} users who don’t follow back.\n`);
}

// === Phase 2: Unfollow them ===
async function unfollowNonFollowers() {
  let totalUnfollowed = 0;
  let pauseCounter = 0;

  for (const user of usersNotFollowingBack) {
    try {
      await fetch(unfollowUserUrl(user.id), {
        method: "POST",
        headers: {
          "content-type": "application/x-www-form-urlencoded",
          "x-csrftoken": csrftoken
        },
        mode: "cors",
        credentials: "include"
      });
    } catch (error) {
      console.error(`Error unfollowing ${user.username}:`, error);
      continue;
    }

    console.log(`Unfollowed: ${user.username} (${++totalUnfollowed}/${usersNotFollowingBack.length})`);

    await sleep(Math.floor(2000 * Math.random()) + 4000); // 4–6 sec delay
    if (++pauseCounter >= 5) {
      console.log("Sleeping 5 minutes to avoid temp block...");
      pauseCounter = 0;
      await sleep(5 * 60 * 1000);
    }
  }

  console.log("All unfollows complete!");
}

// === Run both phases ===
async function runUnfollowScript() {
  await collectNonFollowers();
  await unfollowNonFollowers();
}

runUnfollowScript();
```
# Unfollow Everyone
Just a basic script to unfollow everyone, it will take some time to run. May need to sit overnight.

```Javascript
// Helper to get cookie by name
const getCookie = name => {
  const c = `; ${document.cookie}`;
  const parts = c.split(`; ${name}=`);
  return parts.length === 2 ? parts.pop().split(';')[0] : null;
};

// Sleep helper
const sleep = ms => new Promise(res => setTimeout(res, ms));

const csrftoken = getCookie("csrftoken");
const ds_user_id = getCookie("ds_user_id");

// Fetch all followees recursively
async function fetchFollowees(after = null, acc = []) {
  const variables = {
    id: ds_user_id,
    include_reel: true,
    fetch_mutual: false,
    first: 24,
    after: after
  };

  const url = `https://www.instagram.com/graphql/query/?query_hash=3dec7e2c57367ef3da3d987d89f9dbc8&variables=${encodeURIComponent(JSON.stringify(variables))}`;

  try {
    const res = await fetch(url);
    const json = await res.json();
    const edges = json?.data?.user?.edge_follow?.edges || [];
    const pageInfo = json?.data?.user?.edge_follow?.page_info;

    acc.push(...edges.map(e => e.node.id));

    if (pageInfo?.has_next_page) {
      await sleep(1000);
      return fetchFollowees(pageInfo.end_cursor, acc);
    }
  } catch (err) {
    console.error("Fetch error:", err);
  }

  return acc;
}

// Unfollow users with delay and cooldowns
async function unfollowUsers(userIds) {
  for (let i = 0; i < userIds.length; i++) {
    const id = userIds[i];

    try {
      await fetch(`https://www.instagram.com/web/friendships/${id}/unfollow/`, {
        method: "POST",
        headers: { "x-csrftoken": csrftoken },
        credentials: "include"
      });
    } catch (err) {
      console.error(`Failed to unfollow user ${id}:`, err);
    }

    await sleep(2000); // 2 sec between unfollows

    // Pause every 10 unfollows
    if ((i + 1) % 10 === 0) {
      console.log(`Pausing for 5 minutes after ${i + 1} unfollows...`);
      await sleep(5 * 60 * 1000);
    }
  }
}

// Main execution
(async () => {
  const users = await fetchFollowees();
  if (users.length === 0) {
    console.log("No users to unfollow.");
    return;
  }
  await unfollowUsers(users);
  console.log(`Unfollowed ${users.length} users.`);
})();
```
