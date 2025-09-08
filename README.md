#include <iostream>
#include <string>
#include <regex>
#include <curl/curl.h>
#include <sqlite3.h>
using namespace std;
static size_t WriteCallback(void* contents, size_t size, size_t nmemb, string* output) {
    size_t totalSize = size * nmemb;
    output->append((char*)contents, totalSize);
    return totalSize;
}
string fetchHTML(const string& url) {
    CURL* curl = curl_easy_init();
    string response;
    if (curl) {
        curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &response);
        curl_easy_setopt(curl, CURLOPT_USERAGENT, "Mozilla/5.0");
        curl_easy_perform(curl);
        curl_easy_cleanup(curl);
    }
    return response;
}
void saveToDB(sqlite3* db, const string& title, const string& url, const string& category) {
    string sql = "INSERT OR IGNORE INTO baiviet (title, url, category, created_at) VALUES (?, ?, ?, datetime('now'));";
    sqlite3_stmt* stmt;

    if (sqlite3_prepare_v2(db, sql.c_str(), -1, &stmt, nullptr) == SQLITE_OK) {
        sqlite3_bind_text(stmt, 1, title.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_bind_text(stmt, 2, url.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_bind_text(stmt, 3, category.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_step(stmt);
    }
    sqlite3_finalize(stmt);
}

int main() {
    string url = "https://www.24h.com.vn/the-thao-c101.html";
    string html = fetchHTML(url);
    sqlite3* db;
    if (sqlite3_open("baibao_24h.db", &db)) {
        cerr << "khong mo duoc CSDL!\n";
        return 1;
    }
    const char* sql_create =
        "CREATE TABLE IF NOT EXISTS baiviet ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "title TEXT,"
        "url TEXT UNIQUE,"
        "category TEXT,"
        "created_at TEXT);";
    sqlite3_exec(db, sql_create, nullptr, nullptr, nullptr);
    regex pattern("<a[^>]*class=\"title_news\"[^>]*href=\"([^\"]+)\"[^>]*>(.*?)</a>");
    smatch matches;
    string::const_iterator searchStart(html.cbegin())
    while (regex_search(searchStart, html.cend(), matches, pattern)){
        string link = matches[1];
        string title = regex_replace(matches[2].str(), regex("<[^>]+>"), "");
        saveToDB(db, title, link, "the thao");
        cout << "luu: " << title << endl;
        searchStart = matches.suffix().first;
    }
    sqlite3_close(db);
    cout << "hoan tat.\n";
    return 0;
}
