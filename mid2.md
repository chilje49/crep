```
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include "sqlite3.h"

// 전체 조회용 콜백 함수
static int callback_select(void* data, int argc, char** argv, char** azColName) {
    (void)data; // 미사용 경고 방지
    for (int i = 0; i < argc; i++) {
        printf("%s: %s | ", azColName[i], argv[i] ? argv[i] : "NULL");
    }
    printf("\n");
    return 0;
}

// 검색용 콜백 함수
static int callback_search(void* data, int argc, char** argv, char** azColName) {
    (void)data;
    printf("[검색 결과 매칭] ");
    for (int i = 0; i < argc; i++) {
        printf("%s: %s   ", azColName[i], argv[i] ? argv[i] : "NULL");
    }
    printf("\n");
    return 0;
}

int main() {
    sqlite3* db;
    char* errMessage = 0;
    int rc;

    // 1. 데이터베이스 파일 열기 (없으면 자동 생성)
    rc = sqlite3_open("school.db", &db);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "데이터베이스를 열 수 없습니다: %s\n", sqlite3_errmsg(db));
        sqlite3_close(db);
        return 1;
    }
    printf("[알림] 데이터베이스 연결 성공!\n\n");

    // 2. 테이블 생성 쿼리
    const char* sql_create = 
        "CREATE TABLE IF NOT EXISTS students ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT, "
        "name TEXT NOT NULL, "
        "age INTEGER, "
        "major TEXT, "
        "gpa REAL);";

    rc = sqlite3_exec(db, sql_create, 0, 0, &errMessage);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "테이블 생성 실패: %s\n", errMessage);
        sqlite3_free(errMessage);
    }

    // 기존 데이터 초기화 (테스트 편의용)
    sqlite3_exec(db, "DELETE FROM students;", 0, 0, 0);

    // 3. 3명 이상의 데이터 삽입 (총 4명)
    const char* sql_insert = 
        "INSERT INTO students (name, age, major, gpa) VALUES "
        "('김철수', 21, '컴퓨터공학', 3.8), "
        "('이영희', 22, '인공지능', 4.0), "
        "('박민수', 23, '전자공학', 3.5), "
        "('이지은', 20, '경영학', 3.9);";

    rc = sqlite3_exec(db, sql_insert, 0, 0, &errMessage);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "데이터 삽입 실패: %s\n", errMessage);
        sqlite3_free(errMessage);
    } else {
        printf("[알림] 4명의 학생 데이터가 성공적으로 삽입되었습니다.\n\n");
    }

    // 4. 전체 학생 목록 조회
    printf("=== [DB] 전체 학생 목록 ===\n");
    const char* sql_select_all = "SELECT * FROM students;";
    rc = sqlite3_exec(db, sql_select_all, callback_select, 0, &errMessage);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "조회 실패: %s\n", errMessage);
        sqlite3_free(errMessage);
    }
    printf("===========================\n\n");

    // 5. 검색 기능 구현 (Prepared Statement를 이용한 안전한 검색)
    // 예시: 이름에 '철수'가 포함된 학생 검색
    const char* searchKeyword = "%철수%";
    printf("=== [검색] 키워드 '%s' 검색 결과 ===\n", "철수");

    const char* sql_search = "SELECT * FROM students WHERE name LIKE ? OR major LIKE ?;";
    sqlite3_stmt* stmt;

    rc = sqlite3_prepare_v2(db, sql_search, -1, &stmt, 0);
    if (rc == SQLITE_OK) {
        // 바인딩 (SQL Injection 방지 및 안전한 파라미터 전달)
        sqlite3_bind_text(stmt, 1, searchKeyword, -1, SQLITE_STATIC);
        sqlite3_bind_text(stmt, 2, searchKeyword, -1, SQLITE_STATIC);

        // 결과 순회 출력
        int found = 0;
        while (sqlite3_step(stmt) == SQLITE_ROW) {
            found = 1;
            int id = sqlite3_column_int(stmt, 0);
            const unsigned char* name = sqlite3_column_text(stmt, 1);
            int age = sqlite3_column_int(stmt, 2);
            const unsigned char* major = sqlite3_column_text(stmt, 3);
            double gpa = sqlite3_column_double(stmt, 4);

            printf("ID: %d | 이름: %s | 나이: %d세 | 전공: %s | 학점: %.1f\n", 
                   id, name, age, major, gpa);
        }
        if (!found) {
            printf("검색 결과가 없습니다.\n");
        }
        sqlite3_finalize(stmt);
    } else {
        fprintf(stderr, "검색 쿼리 준비 실패\n");
    }
    printf("===================================\n");

    // 6. 데이터베이스 연결 종료
    sqlite3_close(db);
    printf("[알림] 데이터베이스 연결을 정상적으로 종료했습니다.\n");

    return 0;
}
```
