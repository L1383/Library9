# Library9
<!DOCTYPE html>
<html lang="fa">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>مدیریت رمان‌ها</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <h2>📚 لیست رمان‌های من</h2>
  <button onclick="logout()" style="background:#dc3545; color:white; padding:0.5rem 1rem; border:none; border-radius:5px;">🚪 خروج</button>

  <div class="novel">
    <input type="text" id="novelInput" placeholder="نام رمان" />
    <select id="categoryInput">
      <option value="فانتزی">فانتزی</option>
      <option value="عاشقانه">عاشقانه</option>
      <option value="علمی‌تخیلی">علمی‌تخیلی</option>
      <option value="ماجراجویی">ماجراجویی</option>
      <option value="ترسناک">ترسناک</option>
      <option value="تاریخی">تاریخی</option>
      <option value="درام">درام</option>
      <option value="جنایی">جنایی</option>
      <option value="معمایی">معمایی</option>
      <option value="طنز">طنز</option>
    </select>
    <input type="number" id="chapterCount" placeholder="تعداد کل فصل‌ها" />
    <input type="number" id="chaptersRead" placeholder="تعداد فصل‌های خوانده‌شده" />
    <input type="number" id="ratingInput" placeholder="نمره از 10" step="0.1" />
    <input type="url" id="linkInput" placeholder="لینک مرجع" />
    <label><input type="checkbox" id="favoriteInput" /> مورد علاقه</label>
    <label><input type="checkbox" id="completedInput" /> تکمیل شده</label>
    <button onclick="addNovel()">➕ افزودن</button>
  </div>

  <div class="sort-controls">
    <label>🔽 مرتب‌سازی:
      <select id="sortOption" onchange="renderNovels()">
        <option value="">-- انتخاب کنید --</option>
        <option value="rating">نمره</option>
        <option value="chaptersRead">فصل خوانده‌شده</option>
        <option value="name">نام</option>
        <option value="favorite">مورد علاقه</option>
        <option value="completed">تکمیل شده</option>
      </select>
    </label>
  </div>

  <div class="export-links">
    <a id="downloadJSON" href="#" download="novels.json">📁 دانلود JSON</a>
    <a id="downloadCSV" href="#" download="novels.csv">📊 دانلود CSV</a>
  </div>

  <input type="text" id="searchBox" placeholder="جستجو..." oninput="renderNovels()" style="width: 100%; padding: 0.5rem; margin-top: 1rem;" />
  <div class="novel-list" id="novelList"></div>

  <script type="module">
    import { auth, db } from './firebase.js';
    import {
      signOut,
      onAuthStateChanged
    } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';
    import {
      collection,
      addDoc,
      getDocs,
      updateDoc,
      deleteDoc,
      doc,
      query,
      where
    } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js';

    let user = null;
    let novels = [];
    let novelDocs = [];

    const input = id => document.getElementById(id).value;
    const checkbox = id => document.getElementById(id).checked;

    onAuthStateChanged(auth, async (u) => {
      if (!u) {
        location.href = "index.html";
      } else {
        user = u;
        await loadNovels();
      }
    });

    async function loadNovels() {
      const q = query(collection(db, "novels"), where("uid", "==", user.uid));
      const snap = await getDocs(q);
      novels = [];
      novelDocs = [];
      snap.forEach(docSnap => {
        novels.push(docSnap.data());
        novelDocs.push(docSnap.id);
      });
      renderNovels();
      updateDownloadLinks();
    }

    async function addNovel() {
      const novel = {
        uid: user.uid,
        name: input("novelInput").trim(),
        category: input("categoryInput"),
        chapters: parseInt(input("chapterCount")) || 0,
        chaptersRead: parseInt(input("chaptersRead")) || 0,
        rating: parseFloat(input("ratingInput")) || null,
        link: input("linkInput").trim(),
        favorite: checkbox("favoriteInput"),
        completed: checkbox("completedInput")
      };
      if (!novel.name) return alert("نام رمان الزامی است");

      await addDoc(collection(db, "novels"), novel);
      await loadNovels();
      clearInputs();
    }

    async function editNovel(index) {
      const id = novelDocs[index];
      const n = novels[index];
      const name = prompt("نام رمان:", n.name);
      if (name === null) return;

      const chapters = prompt("تعداد فصل:", n.chapters);
      const chaptersRead = prompt("خوانده‌شده:", n.chaptersRead);
      const rating = prompt("نمره (از 10):", n.rating);
      const link = prompt("لینک:", n.link);
      const favorite = confirm("آیا مورد علاقه است؟");
      const completed = confirm("تکمیل شده؟");

      const newData = {
        ...n,
        name,
        chapters: parseInt(chapters),
        chaptersRead: parseInt(chaptersRead),
        rating: parseFloat(rating),
        link,
        favorite,
        completed
      };
      await updateDoc(doc(db, "novels", id), newData);
      await loadNovels();
    }

    async function deleteNovel(index) {
      const ok = confirm("حذف شود؟");
      if (!ok) return;
      const id = novelDocs[index];
      await deleteDoc(doc(db, "novels", id));
      await loadNovels();
    }

    function clearInputs() {
      ["novelInput", "chapterCount", "chaptersRead", "ratingInput", "linkInput"].forEach(id => document.getElementById(id).value = "");
      document.getElementById("favoriteInput").checked = false;
      document.getElementById("completedInput").checked = false;
    }

    function renderNovels() {
      const list = document.getElementById("novelList");
      const search = document.getElementById("searchBox").value.toLowerCase();
      const sort = document.getElementById("sortOption").value;

      let filtered = novels.map((n, i) => ({ ...n, realIndex: i })).filter(n => n.name.toLowerCase().includes(search));

      if (sort) {
        if (sort === "name") {
          filtered.sort((a, b) => a.name.localeCompare(b.name));
        } else if (sort === "favorite") {
          filtered.sort((a, b) => (b.favorite ? 1 : 0) - (a.favorite ? 1 : 0));
        } else if (sort === "completed") {
          filtered.sort((a, b) => (b.completed ? 1 : 0) - (a.completed ? 1 : 0));
        } else {
          filtered.sort((a, b) => (b[sort] || 0) - (a[sort] || 0));
        }
      }

      list.innerHTML = "";
      filtered.forEach(novel => {
        const item = document.createElement("div");
        item.className = "novel-item";
        item.innerHTML = `
          <div><strong>${novel.name}</strong> ${novel.favorite ? '⭐' : ''} ${novel.completed ? '✔️ تکمیل' : ''}</div>
          <div>دسته: ${novel.category} | فصل: ${novel.chapters} | خوانده: ${novel.chaptersRead}</div>
          ${novel.rating !== null ? `<div>امتیاز: ${novel.rating}/10</div>` : ''}
          ${novel.link ? `<div><a href="${novel.link}" target="_blank">🌐 لینک</a></div>` : ''}
          <div class="actions">
            <button class="edit" onclick="editNovel(${novel.realIndex})">✏️ ویرایش</button>
            <button onclick="deleteNovel(${novel.realIndex})">🗑️ حذف</button>
          </div>`;
        list.appendChild(item);
      });
    }

    function updateDownloadLinks() {
      const dataJSON = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(novels, null, 2));
      const dataCSV = "data:text/csv;charset=utf-8," + encodeURIComponent(
        [["نام","دسته","فصل","خوانده","نمره","لینک","علاقه","تکمیل"]]
        .concat(novels.map(n =>
          [n.name,n.category,n.chapters,n.chaptersRead,n.rating,n.link,n.favorite ? "بله":"خیر",n.completed ? "بله":"خیر"]
        ))
        .map(row => row.join(",")).join("\n")
      );
      document.getElementById("downloadJSON").href = dataJSON;
      document.getElementById("downloadCSV").href = dataCSV;
    }

    function logout() {
      signOut(auth).then(() => location.href = "index.html");
    }
  </script>
</body>
</html>
