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
*Removes people who don't follow you back.*
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

# Possible bot list / Private accounts.
List Generator for possible bots that follow you and profiles you can't view.

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
let possibleBots = [];
let privateFollowers = [];
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
      const postCount = node.edge_owner_to_timeline_media?.count ?? 0;

      // Add to possible bot list (any user with 3 or fewer posts)
      if (postCount <= 3) {
        possibleBots.push({
          username: node.username,
          id: node.id,
          posts: postCount,
          is_private: node.is_private,
          follows_viewer: node.follows_viewer
        });
      }

      // Add to private followers list (private + follows you)
      if (node.is_private && node.follows_viewer) {
        privateFollowers.push({
          username: node.username,
          id: node.id,
          is_private: true,
          follows_viewer: true
        });
      }
    }

    // Log progress
    console.clear();
    console.log(`Progress: ${fetchedCount}/${totalFollowing} (${Math.round(100 * fetchedCount / totalFollowing)}%)`);

    console.log("\nPossible Bots (≤3 posts):");
    possibleBots.forEach(user => {
      console.log(`Username: ${user.username}, ID: ${user.id}, Posts: ${user.posts}, Private: ${user.is_private}, Follows you: ${user.follows_viewer}`);
    });

    console.log("\nPrivate Followers (private + follows you):");
    privateFollowers.forEach(user => {
      console.log(`Username: ${user.username}, ID: ${user.id}, Private: ${user.is_private}, Follows you: ${user.follows_viewer}`);
    });

    // Sleep to avoid rate limits
    await sleep(Math.floor(400 * Math.random()) + 1000);
    sleepCycle++;

    if (sleepCycle > 6) {
      console.log("Sleeping to avoid Instagram block...");
      await sleep(10000);
      sleepCycle = 0;
    }
  }

  // Save results as JSON files
  const saveAsJson = (data, filename) => {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = filename;
    link.click();
  };

  saveAsJson(possibleBots, "IG_PossibleBots.json");
  saveAsJson(privateFollowers, "IG_PrivateFollowers.json");

  console.log("Script completed. Data saved.");
}

startScript();
```

# Auto like posts on profile
Will take a while to run it has to slowly like posts like a human so IG won't throttle your account.

```javascript
async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function likeCurrentPost() {
  await sleep(800);

  let likeButton = [...document.querySelectorAll('svg[aria-label]')]
    .find(svg => svg.getAttribute('aria-label') === 'Like' || svg.getAttribute('aria-label') === 'Unlike');

  if (!likeButton) {
    // fallback: click coordinate (630, 400)
    const el = document.elementFromPoint(630, 400);
    if (el) {
      el.dispatchEvent(new MouseEvent('click', {
        bubbles: true,
        cancelable: true,
        view: window,
        clientX: 630,
        clientY: 400
      }));
      console.log('Clicked like button at coordinate 630x400');
      await sleep(1500);
      return;
    } else {
      console.log('Like button not found and no element at coordinate');
      return;
    }
  }

  if (likeButton.getAttribute('aria-label') === 'Like') {
    likeButton.parentElement.click();
    console.log('Post liked!');
    await sleep(1500);
  } else {
    console.log('Post already liked.');
  }
}

async function closeModal() {
  document.dispatchEvent(new KeyboardEvent('keydown', {
    key: 'Escape',
    keyCode: 27,
    code: 'Escape',
    bubbles: true,
  }));
  await sleep(1000);
}

function getAllPosts() {
  return Array.from(document.querySelectorAll('a'))
    .filter(a => a.href.match(/\/(p|reel)\//));
}

async function likeAllPostsWithBatchScroll(batchSize = 6) {
  const processedPosts = new Set();
  let noNewPostsCounter = 0;

  while (true) {
    // Get all posts currently loaded
    const posts = getAllPosts();
    // Filter out posts already processed
    const newPosts = posts.filter(p => !processedPosts.has(p.href));

    if (newPosts.length === 0) {
      noNewPostsCounter++;
      if (noNewPostsCounter >= 3) {
        console.log('No new posts loaded after scrolling. Ending script.');
        break;
      }
    } else {
      noNewPostsCounter = 0; // reset if found new posts
    }

    // Process posts in batches
    for (let i = 0; i < newPosts.length; i += batchSize) {
      const batch = newPosts.slice(i, i + batchSize);

      for (const post of batch) {
        processedPosts.add(post.href);
        console.log(`Opening post: ${post.href}`);
        post.click();
        await sleep(1000);

        await likeCurrentPost();
        await closeModal();

        await sleep(1000);
      }

      // Scroll after each batch except last batch if no more posts
      if (i + batchSize < newPosts.length || noNewPostsCounter < 3) {
        window.scrollBy(0, window.innerHeight);
        console.log('Scrolled down after batch of posts.');
        await sleep(2000);
      }
    }

    // Small scroll if no new posts or after batch to load more posts
    window.scrollBy(0, window.innerHeight);
    await sleep(2000);
  }

  console.log('All posts processed.');
}

likeAllPostsWithBatchScroll(6);
```
