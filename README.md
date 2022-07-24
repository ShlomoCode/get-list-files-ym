# get-list-files-ym

**קבלת רשימת קבצים בתקיה/תקיות**

```javascript
const axios = require('axios');
axios.defaults.baseURL = 'https://www.call2all.co.il/ym/api/';
const YEMOT_NUMBER = '0773130000';
const YEMOT_PASSWORD = '123456';

/**
 * @return {String} - The token
 */
async function getToken() {
    const response = await axios.post('Login', {
        username: YEMOT_NUMBER,
        password: YEMOT_PASSWORD,
    });
    return response.data.token;
}

/**
 * @param {String} token - Yemot token
 * @param {String} path - folder path
 * @return {Promise<object>} - folder data
 */
async function getFolder(path, token) {
    const { data } = await axios.get('GetIVR2Dir', { params: { token, path } });
    if (data.responseStatus !== 'OK') throw new Error('Yemot error: ' + data.message);
    return data;
}

/**
 *
 * @param {Array|String} begin - Folder(s) to start from
 * @param {Array} exclusionsWords - Words to exclude folders that contain them
 * @param {String} token - Yemot token
 * @returns
 */
async function getLists(begin, exclusionsWords, token = `${YEMOT_NUMBER}:${YEMOT_PASSWORD}`) {
    const lists = { files: [], dirs: [], ini: [] };
    let nextFolders = Array.from(begin);
    while (nextFolders.length) {
        const promises = nextFolders.map((path) => {
            return getFolder(path, token);
        });
        const responses = await Promise.all(promises);
        let localNextFolders = [];
        responses.forEach((folder) => {
            lists.files.push(...folder.files);
            lists.dirs.push(...folder.dirs);
            lists.ini.push(...folder.ini);
            const exclusionsRegex = new RegExp(exclusionsWords.join('|'), 'g');
            const nextFoldersFiltered = folder.dirs.filter((dir) => !exclusionsRegex.test(dir.name)).map((path) => path.what);
            localNextFolders = [...localNextFolders, ...nextFoldersFiltered];
        });
        nextFolders = localNextFolders;
    }
    return lists;
}
```

**דוגמה למימוש - קבלת רשימת הקבצים מתקיות: 1, 2, הדפסת כמות התקיות, כמות קבצים, וכמות קבצי ini, ובנוסף הדפסה של רשימת נתיבי הקבצים ממוינים לפי הנתיב:**

```javascript
(async () => {
    const token = await getToken();
    const lists = await getLists(['1', '2'], ['Log', 'Trash'], token);
    for (const listName in lists) {
        if (Object.hasOwnProperty.call(lists, listName)) {
            console.log(`${listName} count: ${lists[listName].length}`);
        }
    }
    console.log(lists.files.map((file) => file.what).sort());
})();
```

:כמובן שניתן

- לסרוק תקיה אחת, כולל למשל את הראשי
- לא להחריג את תקיות לוג וסל מחזור

אני לא רואה סיבה לא להשתמש בטוקן אלא במספר מערכת וסיסמה אבל אם לא מעבירים ארגומנט טוקן לפונקציה getLists היא משתמש במספר:סיסמה.
