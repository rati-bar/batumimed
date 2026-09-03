# ბათუმის სამედიცინო აკადემია — სისტემის სპეციფიკაცია

ეს დოკუმენტი აღწერს `wireframes/` საქაღალდეში არსებული სამი პორტალის სრულ ლოგიკას იმ დეტალიზაციით, რომ ახალმა სესიამ შეძლოს სისტემის თავიდან აგება ვაერფრეიმების წაკითხვის გარეშე.

| ფაილი | პორტალი | ეკრანი |
| --- | --- | --- |
| `wireframes/admin.html` | ადმინისტრირება | 21 |
| `wireframes/teacher.html` | მასწავლებელი | 7 |
| `wireframes/student.html` | პროფესიული სტუდენტი | 13 |
| `BMA_DB_Tables_CRUD.xlsx` | 36 ცხრილი · 55 FK · 17 enum | — |
| `BMA_Entities_CRUD.xlsx` | 38 ენტიტეტი · 55 კავშირი | — |

დოკუმენტების ორივე გენერატორი ერთ წყაროზეა (`bma_data.py`), ამიტომ სქემის ცვლილება ერთ ადგილას ხდება.

---

## 1. ფუნდამენტური წესები — რასაც სისტემა **არ** აკეთებს

ეს წესები რეგულირებული პროფესიული განათლების მოდელიდან გამომდინარეობს და ყველა ეკრანზე ვრცელდება. მათი დარღვევა სისტემის ხელახლა აგებისას ყველაზე ხშირი შეცდომაა.

1. **სემესტრი არ არსებობს.** სასწავლო პროცესი იზომება **სასწავლო წლებით** და **კვირებით**. პროგრამის `durationYears` განსაზღვრავს წლების რაოდენობას და კვირების ჯამს — ცალკე „სასწავლო პერიოდის" ან „პერიოდის კვირების" ობიექტი **არ არსებობს**.
2. **ქულა არ არსებობს.** შეფასება **ბინარულია**: `confirmed` (დადასტურდა) / `not_confirmed` (ვერ დადასტურდა). არანაირი პროცენტი, ქულა, GPA, საშუალო.
3. **მტკიცებულება არ არსებობს.** არც ერთ პორტალში არ არის სვეტი, ველი ან ატვირთვა „მტკიცებულებისთვის".
4. **დასწრება ბინარულია** — `present` / `absent`. სტატუსი „დააგვიანა" ამოღებულია.
5. **ბრძანება არაფრის წინაპირობა არ არის.** ბრძანება მხოლოდ **აიტვირთება და იკითხება**; სისტემაში სხვა ჩანაწერს არ ებმევა (არც ჩარიცხვას, არც სტატუსის შეჩერებას, არც გადაბარებას). რეგისტრში ქვეყნდება **მხოლოდ ზოგადი ბრძანებები**. ხელმოწერის სვეტი არ არსებობს.
6. **ფიზიკური წიგნები სისტემაში არ ინახება** — გარე პლატფორმის კატალოგის ბმულია (`PRINTED_CATALOG_URL`). ბიბლიოთეკა მართავს **მხოლოდ ელექტრონულ დოკუმენტებს**.
7. **ყველა მომხმარებელი ერთ ცხრილშია** (`usersData`). ინტერფეისს განსაზღვრავს **როლი**. ცალკე `students` / `teachers` / `admin_users` ობიექტები არ არსებობს.
8. **მასალა ებმევა მოდულს, არა ცალკეულ საათს.**
9. **განრიგში ავტომატური განაწილება არ არსებობს** — მხოლოდ ხელით განთავსება და კვირის გასუფთავება.
10. **სტუდენტური ბარათი არ არსებობს.**

### მოხსნილი ენტიტეტები — არ დაამატო თავიდან

ფილიალი · სწავლის შედეგების ცალკე რეესტრი (`outcomeResults`) · მეცადინეობის მასალა · სასწავლო პერიოდები და პერიოდის კვირები · შეფასების ინსტრუმენტები · უწყისები · შუალედური შეფასება · ინდივიდუალური სასწავლო გეგმა (ISP) · აპელაციები · კალენდარული გეგმა · მოდულის შაბლონები · ანგარიშები (შენახული რეპორტები) · აუდიტის ჟურნალის ეკრანი (ცხრილი რჩება, ეკრანი — არა).

---

## 2. მონაცემთა მოდელი

### 2.1 ცენტრალური ჯაჭვი — საათების ერთადერთი წყარო

ეს ჯაჭვი სისტემის გული. **ყველაფერი** — განრიგი, ჟურნალი, დასწრება, შეფასება, გადაბარება — აქედან გამომდინარეობს. არსად არ დუბლირდება.

```
programsData (პროგრამა, durationYears)
   └── intakePrograms (IP)          ← პროგრამა მიღებაზე
         └── intakeModules (IM)     ← მოდული რეესტრიდან + ლექტორი
               └── intakeModuleWeeks (IW)   ← კვირა × {contactH, selfH, assessH}
                     ├── ejSessions()       → საათების გენერაცია
                     └── schedDemand()      → განრიგის მოთხოვნა
```

**საათების გენერაცია** (`ejSessions(code)`) — ID-ები დეტერმინისტულია:

```js
for (const w of imWeeks(im.id)) {
  for (let k = 1; k <= Math.round(w.contactH); k++)
    push({ id: `${code}-W${w.week}-L${k}`, week: w.week, kind: 'lecture',    no: k });
  for (let k = 1; k <= Math.round(w.assessH); k++)
    push({ id: `${code}-W${w.week}-A${k}`, week: w.week, kind: 'assessment', no: k });
}
```

მაგალითი: `MOD-101-W4-L3` — ბიოქიმიის მე-4 კვირის 3-ე ლექცია; `MOD-101-W9-A2` — მე-9 კვირის 2-ე შეფასების საათი. დასწრება (`ejAtt`) და შეფასება (`ejOut`) ამ ID-ებზე ინდექსირდება: `ejAtt[sessionId][studentId] = 'present' | 'absent'`.

**`selfH` (დამოუკიდებელი სამუშაო) საათს არ აგენერირებს** — არც განრიგში ჩანს, არც ჟურნალში. ის მხოლოდ ჯამებში ითვლება.

### 2.2 ცხრილები

**საცნობარო და კონფიგურაცია**
| ცხრილი | გასაღები | ძირითადი ველები | CRUD |
| --- | --- | --- | --- |
| `venuesData` | `id` | `code` (uniq), `type`, `capacity`, `active` | სრული |
| `slotTemplates` | `id` | `ordinal`, `startsAt`, `endsAt` | მხოლოდ ნახვა (I–VII ფიქსირებული) |
| `programsData` | `id` | `name`, `nameEn`, `type`, `nqf`, `qualification`, `durationYears`, `totalCredits`, `status` | სრული |
| `intakesData` | `id` | `name` (uniq), `season`, `year` | სრული |
| `moduleRegistry` | `id` | `title` (uniq), `type` (`theoretical` \| `practical`) | სრული |

**მიღების სტრუქტურა**
| ცხრილი | FK | ველები | შენიშვნა |
| --- | --- | --- | --- |
| `intakePrograms` | `intakeId`, `programId` | — | რედაქტირება არ არის — იშლება და ემატება |
| `intakeModules` | `ipId`, `moduleId`, `teacherId` | — | **პრაქტიკულ მოდულზე `teacherId = null`** |
| `intakeModuleWeeks` | `imId` | `week`, `contactH`, `selfH`, `assessH` | სრული CRUD |
| `modulePrereqs` | `prereqImId`, `dependentImId` | — | ციკლი და თვითწინაპირობა იბლოკება |

**პარტნიორები**
| ცხრილი | FK | ველები |
| --- | --- | --- |
| `partnersData` | — | `name`, `kind` (`dual` \| `cooperative`), `address`, `slots`, `active` |
| `partnerTeachers` | `partnerId` | `name`, `spec`, `phone`, `active` |
| `practicePlacements` | `imId`, `studentId`, `partnerId`, `partnerTeacherId` | — |

**წვდომა და ადამიანები**
| ცხრილი | ველები |
| --- | --- |
| `rolesData` | `id`, `label`, `scope`, `ui` (`admin` \| `teacher` \| `student`), `predefined`, `special` |
| `rolePermissions` | `[roleId][resource] = {c,r,u,d}` — 23 რესურსი × 4 = 92 ბიტი როლზე |
| `usersData` | `role`, `id`, `name`, `email`, `phone`, `status` + როლისთვის სპეციფიკური ველები |
| `studentFilesData` | `studentId`, `name`, `uploadedOn` |

`usersData`-ს როლისთვის სპეციფიკური ველები:
- **student**: `group`, `programId`, `intakeId`, `studyYear`, `balance`, `hasSen`, `socialStatus`, `legalRep*`, `suspendedOn`, `suspensionReason`, `suspensionExpiresOn`, `terminationReason`
- **teacher**: `implementerType`, `spec`, `modules[]`, `workStartedOn`, `qualification`
- **admin**: `dept`

დამხმარე ფუნქციები:
```js
const usersOf   = r => usersData.filter(u => u.role === r);
const roleUi    = r => (rolesData.find(x => x.id === r) || {}).ui || 'admin';
const userUi    = u => roleUi(u.role);
const studentsAll = () => usersOf('student');
const teachersAll = () => usersOf('teacher');
const adminsAll   = () => usersData.filter(u => userUi(u) === 'admin');
```

**სასწავლო პროცესი**
| ცხრილი | გასაღები | ველები |
| --- | --- | --- |
| `groupsData` | `id` | `name`, `programId`, `intakeId`, `studyYear`, `students`, `active` |
| `modulesData` | `code` | `name`, `moduleId`, `groupId`, `programId`, `intakeId`, `teacherId`, `regNo`, `credits`, `startsOn`, `endsOn` |
| `schedPlacements` | `id` | `week`, `day`, `slot`, `venueId`, `groupId`, `imId` |
| `libraryData` | `isbn` | `title`, `author`, `cat`, `moduleCode`, `type`, `url` |
| `moduleMaterials` | `id` | `moduleCode`, `type`, `title`, `source`, `libIsbn?` |

**შეფასება**
| ობიექტი | ფორმა |
| --- | --- |
| `ejAtt` | `{ [sessionId]: { [studentId]: 'present' \| 'absent' } }` |
| `ejOut` | `{ [sessionId]: { [studentId]: 'confirmed' \| 'not_confirmed' } }` |
| `reassessments` | `studentId`, `moduleCode`, `outcomeNo`, `imId`, `groupId`, `attemptNo`, `feePaid`, `scheduledOn`, `dueOn`, `status` |
| `moduleRepeats` | `studentId`, `moduleCode`, `outcomeNo`, `imId`, `failedInGroupId`, `repeatWithGroupId`, `decidedOn`, `status` |

**ადმინისტრირება და კონტენტი**
`auditLog` · `ordersData` · `paymentsData` · `appsData` · `newsAdminData` · `surveysAdminData` · `jobsAdminData` · `clubsAdminData` · `clubEvents` · `clubsData.joined`

### 2.3 ჩამონათვალები (enum)

| enum | ველი | მნიშვნელობები |
| --- | --- | --- |
| `module_type` | `moduleRegistry.type` | `theoretical`, `practical` |
| `session_kind` | სესიის სახეობა | `lecture`, `assessment` |
| `attendance_status` | `ejAtt` | `present`, `absent` |
| `outcome_result` | `ejOut` | `confirmed`, `not_confirmed` |
| `student_status` | `usersData.status` (student) | `active`, `suspended`, `terminated` |
| `teacher_status` | `usersData.status` (teacher) | `active`, `inactive` |
| `program_status` | `programsData.status` | `active`, `phasing_out`, `cancelled` |
| `program_type` | `programsData.type` | `basic_vocational`, `secondary_vocational`, `higher_vocational`, `preparation`, `retraining` |
| `implementer_type` | `usersData.implementerType` | `vocational_teacher` |
| `venue_type` | `venuesData.type` | `auditorium`, `cabinet_lab`, `simulation_lab`, `pharmaco_chem_lab`, `dental_polyclinic`, `partner_workplace` |
| `partner_kind` | `partnersData.kind` | `dual`, `cooperative` |
| `reassessment_status` | `reassessments.status` | `scheduled`, `in_progress`, `confirmed`, `failed` |
| `repeat_status` | `moduleRepeats.status` | `pending`, `in_progress`, `confirmed`, `failed` |
| `material_type` | `moduleMaterials.type` | `presentation`, `document`, `video`, `link`, `library`, `other` |
| `payment_status` | `paymentsData.status` | `paid`, `partial`, `overdue`, `pending` |
| `application_status` | `appsData.status` | `pending`, `approved`, `rejected` |
| `crud_op` | `rolePermissions.op` | `c`, `r`, `u`, `d` |

სტატუსის ლეიბლები `*_LBL` მუდმივებშია, ფორმატით `{key: [label, badgeClass]}` ან `{key: label}`.
დამატებით: `SUSPENSION_LBL` (5 მიზეზი), `TERMINATION_LBL` (9 მიზეზი), `SOCIAL_LBL` (9 კატეგორია) — ეს ჩამონათვალები enum-ებად არ არის რეგისტრირებული, თუმცა UI-ში select-ებად ჩანს.

---

## 3. ბიზნეს-წესები

### 3.1 გადაბარება ≠ მოდულის ხელახლა გავლა

ეს **ორი განსხვავებული პროცესია** და მათი აღრევა კრიტიკული შეცდომაა.

**📝 გადაბარება** (`reassessments`)
- გულისხმობს **მხოლოდ შეფასების საათების** თავიდან გავლას, რომ ლექტორმა დაუდასტუროს მოდული
- სტუდენტი **რჩება საკუთარ ჯგუფში**; ჯგუფი არ იცვლება
- მაქსიმუმ **2 ცდა** (`REASSESS_MAX_ATTEMPTS = 2`): **I უფასო, II ფასიანი** (`RETAKE_FEE = 120` ₾)
- II ცდა **ვერ ინახება** `feePaid === true`-ს გარეშე
- თუ მოდული სხვა მოდულის **წინაპირობაა** → `dueOn = TODAY + 1 კვირა`; სხვა შემთხვევაში `dueOn = null` (ვადა შეუზღუდავია)
- გვერდზე ჩანს, კონკრეტულად რომელი კვირის შეფასების საათებს გაივლის და ჯამი

**🔁 მოდულის ხელახლა გავლა** (`moduleRepeats`)
- **მთელი მოდულის** გავლა **სხვა ჯგუფთან ერთად**
- იხსნება **მხოლოდ მაშინ**, როცა `reassessExhausted()` → `true`:
  ```js
  const reassessExhausted = (sid, code, no) => {
    const l = reassessOf(sid, code, no);
    return l.length >= REASSESS_MAX_ATTEMPTS && l.every(x => x.status === 'failed');
  };
  ```
- ვადამდელი მცდელობა **იბლოკება** ტოსტით
- ინახება ორი ჯგუფი: `failedInGroupId` (საკუთარი) და `repeatWithGroupId` (მასპინძელი)
- `in_progress` სტატუსზე სტუდენტი **გამოჩნდება მასპინძელ ჯგუფშიც** ნიშნულით „ხელახლა გადის მოდულს" და საკუთარი ჯგუფის მითითებით (`repeatsInGroup`)

გადასვლის სრული ციკლი: `ვადამდელი 🔁 → ბლოკი` → `გადაბარება №1 failed` → `№2 იქმნება ფასიანად` → `გადახდის გარეშე → ბლოკი` → `feePaid + failed` → `🔁 იხსნება` → `ჯგუფის გარეშე → ბლოკი` → `შენახვა → სტუდენტი მასპინძელ ჯგუფში`.

### 3.2 პრაქტიკული მოდული

თუ `moduleRegistry.type === 'practical'`:
- ცხრილის შედგენისას **ლექტორი არ ეთითება** (`intakeModules.teacherId = null`); განრიგში ლექტორის კონფლიქტი არ მოწმდება
- ჯგუფი **იშლება** — თითოეულ სტუდენტს ცალკე ენიჭება **პარტნიორი ორგანიზაცია + ამავე ორგანიზაციის მასწავლებელი** (`practicePlacements`)
- სხვა ორგანიზაციის მასწავლებლის არჩევა **იბლოკება** (CHECK წესი)
- ღილაკი 🏥 კლინიკები ხსნის განაწილების გვერდს

### 3.3 განრიგი

- **მოთხოვნა ავტომატურად გენერირდება**: `schedDemand(week)` = აქტიური ჯგუფები × მიღების მოდულები × კვირის `contactH` − უკვე განთავსებული
- ბადე აიწყობა **სლოტი × აუდიტორია** — ერთ საათზე იმდენი სლოტია, რამდენი **აქტიური** აუდიტორიაა
- განთავსება **მხოლოდ ხელითაა**: `pickLesson(groupId, imId)` → `placeHere(day, slot, venueId)`
- სამი კონფლიქტი იბლოკება: `spAt()` (აუდიტორია დაკავებულია), `groupBusy()` (ჯგუფს იმ საათზე სხვა გაკვეთილი აქვს), `teacherBusy()` (ლექტორი დაკავებულია — პრაქტიკულზე არ მოწმდება)
- `clearWeek()` შლის მიმდინარე კვირის (და არჩეული ჯგუფის) განთავსებებს
- დატვირთვის ცხრილი ჯგუფების ჭრილში

### 3.4 მოდულის დასრულება და შეფასების გამოკითხვა

```js
moduleFinished(mi)  // მოდულის ყველა საათზე დასწრება და შეფასება შეტანილია
```
შეფასების (გამოკითხვის) ბმული **არ ჩანს** მოდულის დასრულებამდე — არც მასწავლებელთან, არც სტუდენტთან. დაუსრულებელზე ჩანს ჩაკეტილი მდგომარეობა განმარტებით (`surveyBlocked`).

### 3.5 წაშლის ბლოკები

| ობიექტი | წაშლა იბლოკება, თუ |
| --- | --- |
| პროგრამა | მიღებაზეა მიბმული |
| მიღება | მიღებაზეა პროგრამა, ჯგუფი ან სტუდენტი |
| რეესტრის მოდული | გამოყენებულია მიღების მოდულში |
| ჯგუფი | ჯგუფშია სტუდენტი, მოდული ან ხელახლა გავლის ჩანაწერი |
| აუდიტორია | განრიგშია გამოყენებული |
| პარტნიორი ორგანიზაცია | განაწილებაშია სტუდენტი |

ასევე: **დუბლიკატი** დასახელება იბლოკება რეესტრში, აუდიტორიის კოდში, კლუბის დასახელებაში. **წაშლა არ არსებობს**: მომხმარებელი (სტატუსი იცვლება), გადახდა, განაცხადი, გადაბარება, ხელახლა გავლა, აუდიტის ჩანაწერი.

### 3.6 სტატუსის შეჩერება

სტუდენტს შეიძლება შეუჩერდეს სტატუსი: `status = 'suspended'` + `suspendedOn`, `suspensionReason` (`SUSPENSION_LBL`-დან), `suspensionExpiresOn`. შეჩერების ვადის ამოწურვა შეწყვეტის ერთი მიზეზია (`TERMINATION_LBL.suspension_expired`). შეჩერებული სტუდენტი ჟურნალის აქტიურ სიაში არ ჩანს (`ejStudents` ფილტრავს `status === 'active'`).

### 3.7 გადახდის სტატუსი

ავტომატურად გამოითვლება, ხელით არ იცვლება:
```js
p.paid += amount; p.rest = Math.max(0, p.rest - amount);
p.status = p.rest === 0 ? 'paid' : 'partial';
```

---

## 4. პორტალები

### 4.1 ადმინისტრირება — `admin.html`

**ნავიგაცია (ჯგუფებად):**

| ჯგუფი | ეკრანები |
| --- | --- |
| საცნობარო მონაცემები | 🎯 პროგრამები · 📗 მოდულების რეესტრი · 🗂️ მიღებები · 🏥 პარტნიორი ორგანიზაციები · 📅 განრიგი |
| სასწავლო პროცესი | 🎓 პროფესიული სტუდენტები · 👨‍🏫 მასწავლებლები · 👥 ჯგუფები · 📋 ელექტრონული ჟურნალი |
| ფინანსები და განაცხადები | 📜 ბრძანებები · 💳 გადახდები · 📋 განაცხადები |
| კონტენტი | 📖 ბიბლიოთეკა · 📰 სიახლეები · 📝 გამოკითხვები · 🎭 კლუბები · 💼 კარიერა |
| სისტემა | 👤 მომხმარებლები და როლები · ⚙️ პარამეტრები |

**ეკრანების ID-ები:** `home`, `programs`, `registry`, `intakes`, `partners`, `schedule`, `students`, `teachers`, `groups`, `ejournal`, `orders`, `payments`, `applications`, `library`, `news`, `surveys`, `clubs`, `career`, `users`, `settings`, `detail`.

**მიღებების იერარქია** — ეკრანის ცენტრალური ნაწილი:
```
მიღების ქარდები
  └── პროგრამების ქარდები (წინასწარ განსაზღვრული ჩამონათვალიდან)
        └── მოდულების ცხრილი (რეესტრიდან + ლექტორი)
              ├── 🗓️ საათები       → კვირეული განაწილება (CRUD)
              ├── 🏥 კლინიკები     → პრაქტიკის განაწილება (მხოლოდ პრაქტიკულზე)
              └── წინაპირობები     → რომელი მოდული რისი წინაპირობაა
```
**მოდულის დამატებისას საათები არ ეთითება** — ის მხოლოდ რეესტრიდან აირჩევა და ლექტორი მიენიჭება; საათები ცალკე ღილაკით ემატება კვირების მიხედვით.

**ელექტრონული ჟურნალი** — ადმინს ყველა უფლება აქვს: მოდულის და საათის სახეობის ფილტრი, ყველა საათის ჩამონათვალი, დასწრებისა და შეფასების **შეტანა, შეცვლა და მოხსნა**.

**მომხმარებლები და როლები** — ორი ჩანართი. მომხმარებლების ერთიან სიაში ჩიპებით ფილტრი `u.ui`-ს მიხედვით; როლის შეცვლა (`saveUserRole`) მომხმარებელს გადაიტანს შესაბამის სიაში და როლისთვის სპეციფიკურ ველებს **ავტომატურად შეავსებს** (წინააღმდეგ შემთხვევაში `t.modules.join` ჩავარდება). ორი წინასწარ განსაზღვრული როლი არ იშლება; დანარჩენი აიწყობა 23 რესურსის CRUD მატრიცით, სადაც `c/u/d` ავტომატურად მოითხოვს `r`-ს.

**პარამეტრები** — აუდიტორიების მართვა (CRUD).

### 4.2 მასწავლებელი — `teacher.html`

**ეკრანები:** `home`, `modules` (ჩემი სასწავლო მოდულები), `journal`, `reports`, `surveys`, `profile`, `detail`.

**ჟურნალი საათების ჭრილში** — ცენტრალური ლოგიკა:
1. მოდულის არჩევა → საათების სია (`modSessions`), გენერირებული კვირეული განაწილებიდან: კვირა · სახეობა · ნომერი
2. საათზე გადასვლა (`sessionPage`) → სტუდენტების სია
3. **მეცადინეობის საათზე** — დასწრება (`setAtt`, `markAllPresent`)
4. **შეფასების საათზე** — დამატებით **დაუდასტურდა თუ არა შედეგი** (`setSessOut`, ბინარულად)
5. `studentTotals` — სტუდენტის ჯამები რეალური ჩანაწერებიდან

**სასწავლო მასალები** ებმევა **მოდულს** მენიუდან „ჩემი სასწავლო მოდულები" (`materialsPage`); ცალკე მენიუ „სასწავლო მასალები" არ არსებობს. `addLibDocPage` / `saveLibDoc` ამაგრებს **ბიბლიოთეკის ელექტრონულ დოკუმენტს** (`type: 'library'`, ინახავს `libIsbn`-ს).

**გამოკითხვა** ჩაკეტილია `moduleFinished(mi)`-მდე.

**წაშლილი მენიუები:** სწავლის შედეგები, განრიგი, სასწავლო მასალები, უწყისების ქარდი.

### 4.3 პროფესიული სტუდენტი — `student.html`

**ეკრანები:** `profile`, `docs`, `orders`, `payments`, `apps`, `journal`, `sched`, `lib`, `clubs`, `surveys`, `news`, `career`, `detail`.

**ელექტრონული ჟურნალი** — მოდულების სია + **დასწრების აღრიცხვა თითოეულ ლექციაზე**:
- `modSessions(m)` — მოდულის ყველა საათი; `seedAttendance()` აგენერირებს `myAttendance`-ს **`myModules`-ის გამოცხადების შემდეგ** (თუ ადრე გამოიძახება — `myModules is not defined`)
- `attTable(m)` — კვირა · სახეობა · ნომერი · ჩანაწერი: დავესწარი / გავაცდინე / ჯერ არ ჩატარებულა
- ცალკე ქარდი მოდულის ამორჩევით (`renderAttModule`) და იგივე ცხრილი მოდულის გვერდზე
- ყველა ჯამი რეალური ჩანაწერებიდან, არა ხელით ჩაწერილი

**სასწავლო ცხრილი** — გაკვეთილზე დაჭერისას იხსნება დეტალები: **ლექტორი, საათი, აუდიტორია**, ჯგუფი, დღე, თემატიკა, მასალა. `showClassModal(si, di)` იღებს **ინდექსებს** და მონაცემს `weekData`-დან ამოიღებს — inline JSON `onclick`-ში აპოსტროფზე იშლება (`Gray's Anatomy`).

**ბიბლიოთეკა** — მხოლოდ ელექტრონული დოკუმენტები, ძიებით და კატეგორიის ფილტრით; ბეჭდური წიგნებისთვის ბმული `PRINTED_CATALOG_URL`-ზე.

**მოდულის გვერდი** — ცალკე ცხრილი **გადაბარებებით** (`myReassessments`) და ჩანაწერი **ხელახლა გავლის** შესახებ (`myModuleRepeats`).

**განაცხადები** — შაბლონები, მათ შორის **სტატუსის შეჩერება** და აღდგენა.

---

## 5. ტექნიკური კონვენციები

### 5.1 ფაილის სტრუქტურა

თითოეული პორტალი **ერთი თვითკმარი HTML ფაილია** — inline CSS + inline JS, **გარე დამოკიდებულების გარეშე** (არც CDN, არც ფრეიმვორკი, არც შრიფტის ჩამოტვირთვა).

### 5.2 ბრენდის ტოკენები

```css
--navy:#611a56; --blue:#881176; --green:#1E8A5E;
--amber:#C96A00; --red:#C0392B; --bg:#eff8f1;
font-family:'BPG Nino Mtavruli';
```
ბეჯების კლასები: `bd bd-g` (მწვანე) · `bd-r` (წითელი) · `bd-a` (ყვითელი) · `bd-b` (ლურჯი) · `bd-n` (ნეიტრალური) · `bd-p` (იასამნისფერი).

### 5.3 ეკრანების და შიდა გვერდების სისტემა

```js
go(id, el, title)   // .screen / .screen.on გადართვა + ნავიგაციის აქტიური მდგომარეობა
showModal(title, body, footer, size)   // ↳ ირენდერდება screen-detail-ში, pageStack-ით
closeModal()        // ↳ ბრუნდება pageStack-ის წინა მდგომარეობაზე
```
**მოდალები არ არსებობს ცალკე ოვერლეიად** — ყველა „მოდალი" შიდა გვერდია `screen-detail`-ში. `pageStack` უზრუნველყოფს მრავალდონიან ჩაშლას (მიღება → პროგრამა → მოდული → საათები).

### 5.4 UX კლასები

`.page-head` · `.filter-card` / `.filter-row` / `.filter-hint` · `.stabs` / `.stab` (ჩანართები) · `.av` / `.row-id` · `.note` / `.note-a` · `.dhead` / `.dhead-meta` · `.f3` · `.st4` / `.stc` (სტატისტიკის ქარდები) · `.tw` (ცხრილის სქროლ-კონტეინერი) · `.fg` / `.fg2` / `.fl` / `.fi` (ფორმები) · `.btn btn-b/btn-g/btn-r/btn-a/btn-n/btn-o/btn-ghost` + `btn-sm/btn-xs`.

### 5.5 inline `onclick`-ის ხაფანგები

ეს ორი შეცდომა უკვე დაშვებული და გამოსწორებულია — არ გაიმეორო:
1. `${JSON.stringify(x)}` ორმაგბრჭყალიან `onclick`-ში ტეხს ატრიბუტს → გამოიყენე `'${id}'` და გამოიტანე ლოგიკა ცალკე ფუნქციაში (`doRemoveBook`, `saveBook`)
2. inline JSON აპოსტროფის შემცველი ტექსტით (`Gray's Anatomy`) → გადაეცი **ინდექსები** და მონაცემი ამოიღე მასივიდან

### 5.6 რედაქტირების უსაფრთხო წესი

ვაერფრეიმებზე მუშაობისას გამოიყენე Python სკრიპტი `rep(old, new, label)`-ით, რომელიც **ამტკიცებს ზუსტად 1 დამთხვევას** და ფაილს **მხოლოდ ბოლოს ჩაწერს** — ასე ჩავარდნილი ასერცია ფაილს ხელუხლებელს ტოვებს. მასივის/ფუნქციის/ეკრანის მოსაშლელად გამოიყენე დაბალანსებული ფრჩხილების/`<div>`-ების გადამრთველი (`drop_arr`, `drop_fn`, `drop_screen`).

### 5.7 ტესტირება Playwright-ით

```js
import pkg from '/opt/node22/lib/node_modules/playwright/index.js';
const { chromium } = pkg;
const b = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium' });
```
ავთენტიფიკაცია:
- **admin** — `await p.evaluate(() => doLogin())`, ~900ms
- **teacher** — `doLogin()` → შეავსე `otp0..otp5` → `verifyOTP()`, ~1400ms
- **student** — `doLogin()` → `verifyOTP()`, ~1600ms ორივე ეტაპზე

მინიმალური რეგრესია: გადადი **ყველა** `.screen`-ზე და დაადასტურე, რომ `console` და `pageerror` ცარიელია. ეკრანების მოსალოდნელი რაოდენობა: **admin 21 · teacher 7 · student 13**.

---

## 6. აგების რეკომენდებული თანმიმდევრობა

1. **მონაცემთა ჯაჭვი** — `programsData` → `intakesData` → `moduleRegistry` → `intakePrograms` → `intakeModules` → `intakeModuleWeeks`. სანამ ეს არ დგას, არაფერი აზრიანი არ აიგება.
2. **გენერატორები** — `ejSessions()`, `schedDemand()`, `imWeeks()`, `imTotals()`. საათი არსად ხელით არ იწერება.
3. **მომხმარებლების ერთიანი მოდელი** — `usersData` + `rolesData` + `roleUi()`.
4. **ჯგუფები და მოდულის ინსტანციები** — `groupsData`, `modulesData`.
5. **ჟურნალი** — `ejAtt` / `ejOut` სესიის ID-ებზე; შემდეგ მასწავლებლისა და სტუდენტის პროექციები.
6. **განრიგი** — ბადე სლოტი × აუდიტორია, ხელით განთავსება + სამი კონფლიქტის ბლოკი.
7. **გადაბარება, შემდეგ ხელახლა გავლა** — ამ თანმიმდევრობით, გადასვლის გეითით (`reassessExhausted`).
8. **ფინანსები და განაცხადები**, **კონტენტი**, **წვდომების მატრიცა** — ბოლოს.

ყოველი ეტაპის შემდეგ გაუშვი Playwright-ის რეგრესია: ჩამტვრეული რენდერერი ხშირად მხოლოდ სხვა ეკრანზე გადასვლისას ჩნდება.
