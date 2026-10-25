<template lang="pug">
  .download-app.admin-app
    //- ─── Top bar ────────────────────────────────────────────────
    .admin-topbar(v-if="loggedIn")
      .admin-brand
        icon.fa-fw(name="lock")
        span.admin-title PsiTransfer Admin
      .admin-topbar-actions
        button.btn.btn-sm.btn-default(@click="login()", title="Refresh data")
          icon.fa-fw(name="sync-alt")
          |  Refresh
        button.btn.btn-sm.btn-default(@click="logout()", title="Log out")
          icon.fa-fw(name="sign-out-alt")
          |  Logout

    .alert.alert-danger(v-show="error")
      strong
        icon.fa-fw(name="exclamation-triangle")
        |  {{ error }}

    //- ─── Login ──────────────────────────────────────────────────
    .admin-login(v-if='!loggedIn')
      form.admin-login-card(@submit.stop.prevent="login")
        .admin-login-icon
          icon(name="lock", scale="2.2")
        h3 Admin access
        .form-group
          input.form-control.input-lg(type='password', v-model='password', placeholder="Password", autofocus="")
        p.text-danger.admin-login-err(v-show='passwordWrong')
          icon.fa-fw(name="exclamation-triangle")
          |  Access denied!
        button.btn.btn-primary.btn-block.btn-lg(type="submit", :disabled="!password")
          icon.fa-fw(name="sign-in-alt")
          |  Login

    //- ─── Dashboard ──────────────────────────────────────────────
    div(v-if="loggedIn")
      .admin-stats
        .admin-stat
          .admin-stat-icon
            icon(name="folder")
          div
            .admin-stat-value {{ stats.buckets }}
            .admin-stat-label Buckets
        .admin-stat
          .admin-stat-icon
            icon(name="file")
          div
            .admin-stat-value {{ stats.files }}
            .admin-stat-label Files
        .admin-stat
          .admin-stat-icon
            icon(name="hdd")
          div
            .admin-stat-value {{ humanFileSize(stats.size) }}
            .admin-stat-label Total size
        .admin-stat
          .admin-stat-icon
            icon(name="key")
          div
            .admin-stat-value {{ stats.protected }}
            .admin-stat-label Protected

      .admin-tabs
        button.btn.btn-sm(:class="{'btn-primary': view==='files', 'btn-default': view!=='files'}", @click="view='files'")
          icon.fa-fw(name="file")
          |  Files
        button.btn.btn-sm(:class="{'btn-primary': view==='visitors', 'btn-default': view!=='visitors'}", @click="view='visitors'; if(!visitorsLoaded) fetchVisitors()")
          icon.fa-fw(name="users")
          |  Visitors

      //- ─── Files view ───────────────────────────────────────────
      template(v-if="view==='files'")
        .admin-toolbar
          .admin-search
            icon.admin-search-icon.fa-fw(name="search")
            input.form-control.input-sm(type="text", v-model="searchQuery", placeholder="Search SID, file name, e-mail or IP…")
            a.admin-search-clear(v-if="searchQuery", @click="searchQuery=''", title="Clear") ×
          .admin-toolbar-right
            label.admin-check
              input(type="checkbox", v-model="showExpired")
              |  Show expired
            .btn-group.btn-group-sm.admin-sort
              button.btn.btn-default(@click="toggleSort('created')", :class="{active: sortKey==='created'}") Created {{ sortArrow('created') }}
              button.btn.btn-default(@click="toggleSort('size')", :class="{active: sortKey==='size'}") Size {{ sortArrow('size') }}
              button.btn.btn-default(@click="toggleSort('sid')", :class="{active: sortKey==='sid'}") SID {{ sortArrow('sid') }}

        p.text-muted.admin-empty(v-if="!visibleSids.length")
          icon.fa-fw(name="search")
          |  No buckets match your search.
        table.table.table-hover.admin-table(v-else)
          thead
            tr
              th SID
              th Created
              th Downloaded
              th Expire
              th.text-right Size
              th.text-right Actions
          template(v-for="sid in visibleSids")
            tbody(:class="{expanded: expand===sid}", :key="sid")
              tr.bucket(@click="expandView(sid)")
                td
                  icon.admin-chevron.fa-fw(:name="expand===sid ? 'chevron-down' : 'chevron-right'")
                  code.admin-sid {{ sid }}
                  icon.pull-right.admin-lock(name="key", v-if="sum[sid].password", title="Password protected")
                td {{ sum[sid].created | date }}
                td
                  template(v-if="sum[sid].lastDownload") {{ sum[sid].lastDownload | date}}
                  template(v-else) —
                td
                  template(v-if="typeof sum[sid].firstExpire === 'number'") {{ sum[sid].firstExpire | date }}
                  template(v-else) {{ sum[sid].firstExpire }}
                td.text-right {{ humanFileSize(sum[sid].size) }}
                td.text-right.text-muted {{ db[sid].length }} file{{ db[sid].length === 1 ? '' : 's' }}
            tbody.expanded(v-if="expand === sid", :key="sid + '-files'")
              tr.file(v-for="file in visibleFiles(db[sid])", :key="file.key")
                td.admin-file-name
                  icon.fa-fw.text-muted(:name="isImage(file) ? 'image' : 'file'")
                  |  {{ file.metadata.name }}
                td {{+file.metadata.createdAt | date}}
                td
                  template(v-if="file.metadata.lastDownload") {{ +file.metadata.lastDownload | date}}
                  template(v-else) —
                td
                  template(v-if="typeof file.expireDate === 'number'") {{ file.expireDate | date }}
                  template(v-else) {{ file.expireDate }}
                  span.label.label-warning.expired-badge(v-if="file.metadata.expired") expired
                td.text-right {{ humanFileSize(file.size) }}
                td.text-right.admin-actions
                  a.btn.btn-xs.btn-default(v-if="file.url", :href="file.url", target="_blank", title="Download (works even if bucket is password protected or expired)")
                    icon.fa-fw(name="download")
                  |
                  a.btn.btn-xs.btn-default(@click="showData(file)", title="Known info about the uploader")
                    icon.fa-fw(name="info-circle")
                  |
                  a.btn.btn-xs.btn-default(v-if="isImage(file)", @click="showExif(file)", title="Read image EXIF metadata")
                    icon.fa-fw(name="camera")
                  |
                  a.btn.btn-xs.btn-danger(@click="deleteFile(file)", title="Permanently delete this file")
                    icon.fa-fw(name="trash")
        .admin-tablefoot(v-if="visibleSids.length")
          span Showing {{ visibleSids.length }} of {{ stats.buckets }} bucket{{ stats.buckets === 1 ? '' : 's' }}
          span.pull-right Total: {{ humanFileSize(sizeSum) }}

      //- ─── Visitors view ────────────────────────────────────────
      template(v-if="view==='visitors'")
        p.text-muted.admin-empty(v-if="visitorsLoading")
          icon.fa-fw(name="sync-alt")
          |  Loading visitors…
        p.text-muted.admin-empty(v-else-if="!sortedVisitors.length")
          icon.fa-fw(name="users")
          |  No visitors tracked yet.
        table.table.table-hover.admin-table(v-else)
          thead
            tr
              th E-mail
              th TID
              th IP
              th.text-right Up
              th.text-right Files
              th.text-right Visits
              th.text-right Dl
              th Last seen
              th.text-right Actions
          tbody
            tr(v-for="v in sortedVisitors", :key="v.trackerId || v.ip")
              td {{ v.email || '—' }}
              td
                code.admin-sid {{ (v.trackerId || '').slice(0, 8) || '—' }}
              td {{ v.ip || '—' }}
              td.text-right {{ v.uploads }}
              td.text-right {{ v.files }}
              td.text-right {{ v.visits }}
              td.text-right {{ v.downloads }}
              td {{ v.lastSeen | date }}
              td.text-right
                a.btn.btn-xs.btn-default(@click="showVisitorDetail(v)") Details

    //- ─── Uploader info modal ────────────────────────────────────
    .data-modal-backdrop(v-if="dataFile", @click.self="closeData")
      .data-modal
        a.btn.btn-xs.btn-default.data-modal-close(@click="closeData") ×
        h4
          icon.fa-fw(name="info-circle")
          |  Uploader info
        dl.dl-horizontal
          dt IP address
          dd {{ dataFile.metadata.uploaderIp || 'unknown' }}
          dt Location
          dd
            template(v-if="geoLoading") looking up…
            template(v-else-if="geoInfo && geoInfo.private") Private/local network
            template(v-else-if="geoInfo && geoInfo.city") {{ geoInfo.city }}, {{ geoInfo.regionName }}, {{ geoInfo.country }}
            template(v-else-if="geoInfo && geoInfo.error") lookup failed
            template(v-else) —
          dt E-mail
          dd {{ dataFile.metadata.email || '—' }}
          dt Tracker ID
          dd
            code {{ dataFile.metadata.trackerId || dataFile.metadata.fingerprint || '—' }}
          dt Timezone
          dd {{ dataFile.metadata.timezone || 'unknown' }}
          dt ISP
          dd {{ (geoInfo && geoInfo.isp) || '—' }}
          dt Language
          dd {{ (dataFile.metadata.uploaderLang || '').split(',')[0] || 'unknown' }}
          dt Browser
          dd {{ parseUserAgent(dataFile.metadata.uploaderUserAgent).browser }}
          dt OS
          dd {{ parseUserAgent(dataFile.metadata.uploaderUserAgent).os }}

    //- ─── Visitor detail modal ───────────────────────────────────
    .visitor-modal-backdrop(v-if="detailVisitor", @click.self="closeVisitorDetail")
      .data-modal
        a.btn.btn-xs.btn-default.data-modal-close(@click="closeVisitorDetail") ×
        h4
          icon.fa-fw(name="users")
          |  Visitor
        dl.dl-horizontal
          dt E-mail
          dd {{ detailVisitor.info.email || '—' }}
          dt Tracker ID
          dd
            code {{ detailVisitor.info.trackerId || '—' }}
          dt IP
          dd {{ detailVisitor.info.ip || '—' }}
          dt Activity
          dd {{ detailVisitor.info.uploads }} up · {{ detailVisitor.info.files }} files · {{ detailVisitor.info.visits }} visits · {{ detailVisitor.info.downloads }} dl
          dt Browser
          dd {{ parseUserAgent(detailVisitor.info.ua).browser }} — {{ parseUserAgent(detailVisitor.info.ua).os }}
        h5 Files ({{ detailVisitor.files.length }})
        table.table.table-condensed(v-if="detailVisitor.files.length")
          tbody
            tr(v-for="f in detailVisitor.files", :key="f.sid + f.key")
              td {{ f.name }}
              td {{ +f.createdAt | date }}
              td.text-right {{ humanFileSize(f.size) }}
        h5(v-if="detailVisitor.events.length") Events ({{ detailVisitor.events.length }})
        ul.list-group(v-if="detailVisitor.events.length")
          li.list-group-item(v-for="e in detailVisitor.events", :key="e.ts + e.kind + (e.file || '')")
            span.label(:class="e.kind==='upload' ? 'label-success' : (e.kind==='download' ? 'label-info' : 'label-default')")  {{ e.kind }}
            span.visitor-evt  {{ +e.ts | date }} — {{ e.file || e.sid || '' }}

    //- ─── EXIF modal ─────────────────────────────────────────────
    .data-modal-backdrop(v-if="exifFile", @click.self="closeExif")
      .data-modal.exif-modal
        a.btn.btn-xs.btn-default.data-modal-close(@click="closeExif") ×
        h4
          icon.fa-fw(name="camera")
          |  EXIF metadata
        p.text-muted(v-if="exifLoading") Reading image metadata…
        p.text-danger(v-else-if="exifError") {{ exifError }}
        p.text-muted(v-else-if="!exifEntries.length") No EXIF metadata found.
        table.table.table-condensed.table-striped(v-else)
          tbody
            tr(v-for="entry in exifEntries", :key="entry.key")
              th {{ entry.key }}
              td.text-break {{ entry.value }}

</template>


<script>
  import 'vue-awesome/icons/exclamation-triangle';
  import 'vue-awesome/icons/sync-alt';
  import 'vue-awesome/icons/sign-in-alt';
  import 'vue-awesome/icons/sign-out-alt';
  import 'vue-awesome/icons/key';
  import 'vue-awesome/icons/lock';
  import 'vue-awesome/icons/download';
  import 'vue-awesome/icons/trash';
  import 'vue-awesome/icons/info-circle';
  import 'vue-awesome/icons/camera';
  import 'vue-awesome/icons/search';
  import 'vue-awesome/icons/chevron-down';
  import 'vue-awesome/icons/chevron-right';
  import 'vue-awesome/icons/file';
  import 'vue-awesome/icons/image';
  import 'vue-awesome/icons/folder';
  import 'vue-awesome/icons/hdd';
  import 'vue-awesome/icons/users';
  import exifr from 'exifr';


  export default {
    name: 'app',

    data () {
      return {
        db: {},
        sum: {},
        loggedIn: false,
        password: '',
        error: '',
        passwordWrong: false,
        expand: false,
        sizeSum: 0,
        view: 'files',
        searchQuery: '',
        showExpired: true,
        sortKey: 'created',
        sortDir: 'desc',
        dataFile: null,
        geoInfo: null,
        geoLoading: false,
        exifFile: null,
        exifEntries: [],
        exifLoading: false,
        exifError: '',
        visitors: [],
        visitorsLoaded: false,
        visitorsLoading: false,
        detailVisitor: null
      }
    },

    computed: {
      sortedVisitors() {
        return this.visitors.slice().sort((a, b) => (b.lastSeen || 0) - (a.lastSeen || 0));
      },

      // Dashboard totals computed from the whole db (search-independent).
      stats() {
        const sids = Object.keys(this.db);
        let files = 0, size = 0, protectedBuckets = 0;
        sids.forEach(sid => {
          files += this.db[sid].length;
          if (this.sum[sid] && this.sum[sid].password) protectedBuckets++;
          this.db[sid].forEach(f => { size += f.size; });
        });
        return { buckets: sids.length, files, size, protected: protectedBuckets };
      },

      // Buckets matching the current search, in the chosen sort order.
      visibleSids() {
        const q = this.searchQuery.trim().toLowerCase();
        let sids = Object.keys(this.db);

        if (q) sids = sids.filter(sid => this.bucketMatches(sid, q));

        const dir = this.sortDir === 'asc' ? 1 : -1;
        const key = this.sortKey;
        return sids.sort((a, b) => {
          if (key === 'sid') return a.localeCompare(b) * dir;
          const va = (this.sum[a] && this.sum[a][key === 'size' ? 'size' : 'created']) || 0;
          const vb = (this.sum[b] && this.sum[b][key === 'size' ? 'size' : 'created']) || 0;
          return (va - vb) * dir;
        });
      }
    },

    mounted() {
      document.addEventListener('keydown', this.handleKey);
    },

    beforeDestroy() {
      document.removeEventListener('keydown', this.handleKey);
    },

    methods: {
      // Close any open modal with the Escape key.
      handleKey(e) {
        if (e.key !== 'Escape') return;
        if (this.exifFile) return this.closeExif();
        if (this.detailVisitor) return this.closeVisitorDetail();
        if (this.dataFile) return this.closeData();
      },

      bucketMatches(sid, q) {
        if (sid.toLowerCase().includes(q)) return true;
        return (this.db[sid] || []).some(f => {
          const m = f.metadata || {};
          return (m.name && m.name.toLowerCase().includes(q))
            || (m.email && m.email.toLowerCase().includes(q))
            || (m.uploaderIp && m.uploaderIp.toLowerCase().includes(q));
        });
      },

      visibleFiles(bucket) {
        if (this.showExpired) return bucket;
        return bucket.filter(f => !(f.metadata && f.metadata.expired));
      },

      toggleSort(key) {
        if (this.sortKey === key) {
          this.sortDir = this.sortDir === 'asc' ? 'desc' : 'asc';
        } else {
          this.sortKey = key;
          this.sortDir = key === 'sid' ? 'asc' : 'desc';
        }
      },

      sortArrow(key) {
        if (this.sortKey !== key) return '';
        return this.sortDir === 'asc' ? '▲' : '▼';
      },

      logout() {
        this.loggedIn = false;
        this.password = '';
        this.db = {};
        this.sum = {};
        this.expand = false;
        this.sizeSum = 0;
        this.searchQuery = '';
        this.visitors = [];
        this.visitorsLoaded = false;
        this.view = 'files';
        this.error = '';
        this.closeData();
        this.closeExif();
        this.closeVisitorDetail();
      },

      expandView(sid) {
        if(this.expand === sid) return this.expand = false;
        this.expand = sid;
      },

      login() {
        if(!this.password) return;
        const xhr = new XMLHttpRequest();
        xhr.open('GET', 'admin/data.json');
        xhr.setRequestHeader("x-passwd", this.password);
        xhr.onload = () => {
          if(xhr.status === 200) {
            try {
              this.db = JSON.parse(xhr.responseText);
              this.loggedIn = true;
              this.error = '';
              this.passwordWrong = false;
              this.expandDb();
            }
            catch(e) {
              this.error = e.toString();
            }
          } else {
            if(xhr.status === 403) this.passwordWrong = true;
            else this.error = `${xhr.status} ${xhr.statusText}: ${xhr.responseText}`;
          }
        };
        xhr.send();
      },

      expandDb() {
        this.sizeSum = 0;
        Object.keys(this.db).forEach(sid => {
          const bucketSum = {
            firstExpire: Number.MAX_SAFE_INTEGER,
            lastDownload: 0,
            created: Number.MAX_SAFE_INTEGER,
            password: false,
            size: 0
          };
          this.db[sid].forEach(file => {
            bucketSum.size += file.size;
            if(file.metadata._password) {
              bucketSum.password = true;
            }
            if(+file.metadata.createdAt < bucketSum.created) {
              bucketSum.created = +file.metadata.createdAt;
            }
            if(file.metadata.lastDownload && +file.metadata.lastDownload > bucketSum.lastDownload) {
              bucketSum.lastDownload = +file.metadata.lastDownload;
            }
            if(file.metadata.retention === 'one-time') {
              bucketSum.firstExpire = 'one-time';
              file.expireDate = file.metadata.retention;
            }
            else {
              file.expireDate = +file.metadata.createdAt + (+file.metadata.retention * 1000);
              if(bucketSum.firstExpire > file.expireDate) bucketSum.firstExpire = file.expireDate;
            }
          });
          this.sizeSum += bucketSum.size;
          this.$set(this.sum, sid, bucketSum);
        });
      },

      humanFileSize(fileSizeInBytes) {
        let i = -1;
        const byteUnits = [' kB', ' MB', ' GB', ' TB', 'PB', 'EB', 'ZB', 'YB'];
        do {
          fileSizeInBytes = fileSizeInBytes / 1024;
          i++;
        } while(fileSizeInBytes > 1024);
        return Math.max(fileSizeInBytes, 0.00).toFixed(2) + byteUnits[i];
      },

      deleteFile(file) {
        const name = file.metadata.name || file.key;
        if (!window.confirm(`Delete "${name}" permanently? This cannot be undone.`)) return;

        const sid = file.metadata.sid;
        const key = file.key;
        const xhr = new XMLHttpRequest();
        xhr.open('DELETE', `admin/files/${sid}/${key}`);
        xhr.setRequestHeader('x-passwd', this.password);
        xhr.onload = () => {
          if (xhr.status === 204) {
            const bucket = this.db[sid];
            const idx = bucket.findIndex(f => f.key === key);
            if (idx !== -1) bucket.splice(idx, 1);
            if (bucket.length === 0) {
              this.$delete(this.db, sid);
              if (this.expand === sid) this.expand = false;
            }
            this.expandDb();
          } else {
            this.error = `${xhr.status} ${xhr.statusText}: ${xhr.responseText}`;
          }
        };
        xhr.onerror = () => {
          this.error = 'Network error while deleting the file.';
        };
        xhr.send();
      },

      showData(file) {
        this.dataFile = file;
        this.geoInfo = null;
        const ip = file.metadata.uploaderIp;
        if (!ip) return;
        this.geoLoading = true;
        const xhr = new XMLHttpRequest();
        xhr.open('GET', `admin/geoip/${encodeURIComponent(ip)}`);
        xhr.setRequestHeader('x-passwd', this.password);
        xhr.onload = () => {
          this.geoLoading = false;
          try {
            this.geoInfo = JSON.parse(xhr.responseText);
          } catch (e) {
            this.geoInfo = { error: true };
          }
        };
        xhr.onerror = () => {
          this.geoLoading = false;
          this.geoInfo = { error: true };
        };
        xhr.send();
      },

      isImage(file) {
        return !!(file && file.metadata && /^image\//.test(file.metadata.type || ''));
      },

      async showExif(file) {
        this.exifFile = file;
        this.exifEntries = [];
        this.exifError = '';
        this.exifLoading = true;
        try {
          const response = await fetch(file.url);
          if (!response.ok) throw new Error(`Unable to read image (${response.status})`);
          const metadata = await exifr.parse(await response.blob());
          this.exifEntries = Object.keys(metadata || {})
            .sort()
            .map(key => ({ key, value: this.formatExifValue(metadata[key]) }));
        } catch (e) {
          this.exifError = e.message || 'Unable to read image metadata.';
        } finally {
          this.exifLoading = false;
        }
      },

      formatExifValue(value) {
        if (value instanceof Date) return value.toISOString();
        if (Array.isArray(value)) return value.map(item => this.formatExifValue(item)).join(', ');
        if (value && typeof value === 'object') return JSON.stringify(value);
        if (typeof value === 'number') return Number.isFinite(value) ? String(value) : 'unknown';
        return String(value);
      },

      closeExif() {
        this.exifFile = null;
        this.exifEntries = [];
        this.exifError = '';
      },

      closeData() {
        this.dataFile = null;
        this.geoInfo = null;
      },

      fetchVisitors() {
        this.visitorsLoading = true;
        const xhr = new XMLHttpRequest();
        xhr.open('GET', 'admin/visitors.json');
        xhr.setRequestHeader('x-passwd', this.password);
        xhr.onload = () => {
          this.visitorsLoading = false;
          if (xhr.status === 200) {
            try {
              const data = JSON.parse(xhr.responseText);
              this.visitors = data.visitors || [];
              this.visitorsLoaded = true;
            } catch (e) {
              this.error = 'Error parsing visitors: ' + e.message;
            }
          } else {
            this.error = `Visitors: ${xhr.status} ${xhr.statusText}`;
          }
        };
        xhr.onerror = () => { this.visitorsLoading = false; };
        xhr.send();
      },

      showVisitorDetail(v) {
        this.detailVisitor = { info: v, files: [], events: [] };
        if (!v.trackerId) return;
        const xhr = new XMLHttpRequest();
        xhr.open('GET', 'admin/visitors.json?tid=' + encodeURIComponent(v.trackerId));
        xhr.setRequestHeader('x-passwd', this.password);
        xhr.onload = () => {
          try {
            const data = JSON.parse(xhr.responseText);
            this.detailVisitor = {
              info: data.visitor || v,
              files: (data.files || []).slice(0, 200).reverse(),
              events: (data.events || []).slice().reverse()
            };
          } catch (e) {
            this.error = e.message;
          }
        };
        xhr.send();
      },

      closeVisitorDetail() {
        this.detailVisitor = null;
      },

      parseUserAgent(ua) {
        if (!ua) return { browser: 'unknown', os: 'unknown' };

        let os = 'unknown';
        if (/Windows NT 10/.test(ua)) os = 'Windows 10/11';
        else if (/Windows NT 6\.3/.test(ua)) os = 'Windows 8.1';
        else if (/Windows NT 6\.1/.test(ua)) os = 'Windows 7';
        else if (/Windows/.test(ua)) os = 'Windows';
        else if (/Android/.test(ua)) {
          const m = ua.match(/Android (\d+(\.\d+)?)/);
          os = m ? `Android ${m[1]}` : 'Android';
        } else if (/iPhone|iPad|iPod/.test(ua)) {
          const m = ua.match(/OS (\d+[_.]\d+)/);
          os = m ? `iOS ${m[1].replace('_', '.')}` : 'iOS';
        } else if (/Mac OS X/.test(ua)) {
          const m = ua.match(/Mac OS X (\d+[_.]\d+)/);
          os = m ? `macOS ${m[1].replace('_', '.')}` : 'macOS';
        } else if (/Linux/.test(ua)) os = 'Linux';

        let browser = 'unknown';
        let m;
        if ((m = ua.match(/Edg\/([\d.]+)/))) browser = `Edge ${m[1]}`;
        else if ((m = ua.match(/OPR\/([\d.]+)/))) browser = `Opera ${m[1]}`;
        else if ((m = ua.match(/Firefox\/([\d.]+)/))) browser = `Firefox ${m[1]}`;
        else if (/Chrome\//.test(ua) && !/Chromium/.test(ua) && (m = ua.match(/Chrome\/([\d.]+)/))) browser = `Chrome ${m[1]}`;
        else if (/Safari\//.test(ua) && (m = ua.match(/Version\/([\d.]+)/))) browser = `Safari ${m[1]}`;

        return { browser, os };
      },

    },


  }
</script>

<style>
  .admin-app {
    max-width: 1024px;
    margin: 0 auto;
    padding: 0 10px 40px;
  }

  /* ── Top bar ───────────────────────────────────────────── */
  .admin-topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 8px;
    padding: 14px 0 12px;
    margin-bottom: 16px;
    border-bottom: 1px solid #e3e3e3;
  }
  .admin-brand {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 20px;
    color: #2c3e50;
  }
  .admin-title {
    font-weight: 600;
  }
  .admin-topbar-actions {
    display: flex;
    gap: 6px;
  }

  /* ── Login ─────────────────────────────────────────────── */
  .admin-login {
    display: flex;
    justify-content: center;
    padding: 8vh 0;
  }
  .admin-login-card {
    width: 100%;
    max-width: 340px;
    text-align: center;
    background: #fff;
    border: 1px solid #e3e3e3;
    border-radius: 8px;
    padding: 32px 28px;
    box-shadow: 0 6px 24px rgba(0,0,0,0.06);
  }
  .admin-login-icon {
    color: #4a90d9;
    margin-bottom: 8px;
  }
  .admin-login-card h3 {
    margin-top: 4px;
    margin-bottom: 20px;
    font-weight: 600;
  }
  .admin-login-err {
    margin: 8px 0 12px;
  }

  /* ── Stat cards ────────────────────────────────────────── */
  .admin-stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 18px;
  }
  .admin-stat {
    display: flex;
    align-items: center;
    gap: 12px;
    background: #fff;
    border: 1px solid #e7e7e7;
    border-radius: 8px;
    padding: 14px 16px;
  }
  .admin-stat-icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    border-radius: 8px;
    background: #eef4fb;
    color: #4a90d9;
    flex: none;
  }
  .admin-stat-value {
    font-size: 20px;
    font-weight: 600;
    line-height: 1.1;
    color: #2c3e50;
  }
  .admin-stat-label {
    font-size: 12px;
    text-transform: uppercase;
    letter-spacing: .04em;
    color: #9098a1;
  }

  /* ── Tabs ──────────────────────────────────────────────── */
  .admin-tabs {
    display: flex;
    gap: 6px;
    margin-bottom: 14px;
  }

  /* ── Toolbar (search / sort) ───────────────────────────── */
  .admin-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 10px;
  }
  .admin-search {
    position: relative;
    flex: 1 1 260px;
    max-width: 420px;
  }
  .admin-search .admin-search-icon {
    position: absolute;
    left: 10px;
    top: 50%;
    transform: translateY(-50%);
    color: #aab2bd;
  }
  .admin-search input {
    padding-left: 30px;
    padding-right: 26px;
  }
  .admin-search-clear {
    position: absolute;
    right: 8px;
    top: 50%;
    transform: translateY(-50%);
    cursor: pointer;
    color: #999;
    font-size: 16px;
    line-height: 1;
    text-decoration: none;
  }
  .admin-toolbar-right {
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .admin-check {
    margin: 0;
    font-weight: normal;
    color: #555;
    cursor: pointer;
    user-select: none;
  }
  .admin-sort .btn.active {
    background: #4a90d9;
    color: #fff;
    border-color: #4a90d9;
  }

  /* ── Tables ────────────────────────────────────────────── */
  .admin-table {
    background: #fff;
    border: 1px solid #e7e7e7;
    border-radius: 8px;
    overflow: hidden;
    margin-bottom: 8px;
  }
  .admin-table thead th {
    border-bottom: 2px solid #e7e7e7;
    color: #7a828c;
    font-size: 12px;
    text-transform: uppercase;
    letter-spacing: .03em;
  }
  .bucket {
    cursor: pointer;
  }
  .admin-chevron {
    color: #aab2bd;
    margin-right: 6px;
  }
  .admin-sid {
    background: #f2f5f8;
    color: #39506b;
    padding: 1px 6px;
    border-radius: 4px;
    font-size: 12px;
  }
  .admin-lock {
    color: #e0a800;
    margin-top: 3px;
  }
  .expanded {
    background: #f7fafd;
  }
  .expanded .bucket td {
    font-weight: 600;
  }
  tbody.expanded .file td {
    border-top: 1px solid #edf1f5;
  }
  .admin-file-name {
    max-width: 320px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .admin-actions .btn + .btn {
    margin-left: 2px;
  }
  .expired-badge {
    margin-left: 6px;
    vertical-align: middle;
  }
  .admin-tablefoot {
    padding: 8px 4px;
    color: #7a828c;
    font-size: 13px;
    border-top: 1px solid #edf1f5;
  }
  .admin-empty {
    padding: 36px 0;
    text-align: center;
    font-size: 15px;
  }

  /* ── Modals ────────────────────────────────────────────── */
  .data-modal-backdrop,
  .visitor-modal-backdrop {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    animation: admin-fade .12s ease-out;
  }
  @keyframes admin-fade {
    from { opacity: 0; }
    to   { opacity: 1; }
  }
  .data-modal {
    background: #fff;
    border-radius: 8px;
    padding: 20px 24px;
    max-width: 480px;
    width: 90%;
    position: relative;
    box-shadow: 0 10px 40px rgba(0,0,0,0.3);
  }
  .data-modal h4 {
    margin-top: 0;
    margin-bottom: 16px;
  }
  .visitor-modal-backdrop .data-modal {
    max-width: 640px;
    max-height: 90vh;
    overflow-y: auto;
  }
  .exif-modal {
    max-height: 90vh;
    overflow-y: auto;
  }
  .exif-modal th {
    width: 35%;
  }
  .data-modal-close {
    position: absolute;
    top: 10px;
    right: 10px;
  }
  .dl-horizontal dt {
    width: 130px;
  }
  .dl-horizontal dd {
    margin-left: 140px;
  }
  .text-break {
    word-break: break-all;
  }
  .visitor-evt {
    margin-left: 10px;
    color: #555;
  }

  /* ── Responsive ────────────────────────────────────────── */
  @media (max-width: 640px) {
    .admin-stats {
      grid-template-columns: repeat(2, 1fr);
    }
    .admin-toolbar-right {
      width: 100%;
      justify-content: space-between;
    }
    .admin-file-name {
      max-width: 160px;
    }
  }
</style>
