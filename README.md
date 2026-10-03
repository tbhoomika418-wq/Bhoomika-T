# Bhoomika-T
pet registration system for doctor appointment
/*
 * Pet Care Clinic - Appointment Manager (C)
 * Data is stored in binary files: pets.dat and appts.dat
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define PETS_FILE  "pets.dat"
#define APPTS_FILE "appts.dat"

enum { SCHEDULED, COMPLETED, CANCELLED };
static const char *STATUS_TEXT[] = {"Scheduled", "Completed", "Cancelled"};

typedef struct {
    int  id;
    char name[50];
    char species[30];
    char owner[50];
    char phone[20];
} Pet;

typedef struct {
    int  id;
    int  pet_id;
    char vet[40];
    char date[11];    /* YYYY-MM-DD */
    char time[6];     /* HH:MM      */
    char reason[100];
    int  status;
} Appointment;

/* ---------- Input helpers ---------- */
static void read_line(const char *prompt, char *buf, size_t size) {
    printf("%s", prompt);
    if (!fgets(buf, (int)size, stdin)) { buf[0] = '\0'; return; }
    size_t len = strlen(buf);
    if (len > 0 && buf[len - 1] == '\n') buf[len - 1] = '\0';
    else { int c; while ((c = getchar()) != '\n' && c != EOF); } /* discard overflow */
}

static int read_int(const char *prompt) {
    char buf[32];
    read_line(prompt, buf, sizeof buf);
    return atoi(buf);
}

static void read_nonempty(const char *prompt, char *buf, size_t size) {
    do {
        read_line(prompt, buf, size);
        if (buf[0] == '\0') printf("  This field cannot be empty.\n");
    } while (buf[0] == '\0');
}

/* ---------- Validation ---------- */
static int valid_date(const char *s) {
    int y, m, d;
    if (strlen(s) != 10 || sscanf(s, "%4d-%2d-%2d", &y, &m, &d) != 3) return 0;
    if (s[4] != '-' || s[7] != '-') return 0;
    if (m < 1 || m > 12 || d < 1 || d > 31) return 0;
    return 1;
}

static int valid_time(const char *s) {
    int h, m;
    if (strlen(s) != 5 || s[2] != ':' || sscanf(s, "%2d:%2d", &h, &m) != 2) return 0;
    return h >= 0 && h <= 23 && m >= 0 && m <= 59;
}

static void today_string(char *out) {  /* out must hold 11 bytes */
    time_t now = time(NULL);
    strftime(out, 11, "%Y-%m-%d", localtime(&now));
}

/* ---------- File helpers ---------- */
static int next_id(const char *file, size_t rec_size) {
    FILE *fp = fopen(file, "rb");
    if (!fp) return 1;
    int max_id = 0;
    char *rec = malloc(rec_size);
    while (fread(rec, rec_size, 1, fp) == 1) {
        int id = *(int *)rec;          /* id is always the first field */
        if (id > max_id) max_id = id;
    }
    free(rec);
    fclose(fp);
    return max_id + 1;
}

static int find_pet(int id, Pet *out) {
    FILE *fp = fopen(PETS_FILE, "rb");
    if (!fp) return 0;
    Pet p;
    while (fread(&p, sizeof p, 1, fp) == 1) {
        if (p.id == id) { if (out) *out = p; fclose(fp); return 1; }
    }
    fclose(fp);
    return 0;
}

/* ---------- Features ---------- */
static void add_pet(void) {
    Pet p;
    memset(&p, 0, sizeof p);
    p.id = next_id(PETS_FILE, sizeof(Pet));
    read_nonempty("Pet name: ",    p.name,    sizeof p.name);
    read_nonempty("Species: ",     p.species, sizeof p.species);
    read_nonempty("Owner name: ",  p.owner,   sizeof p.owner);
    read_nonempty("Owner phone: ", p.phone,   sizeof p.phone);

    FILE *fp = fopen(PETS_FILE, "ab");
    if (!fp) { perror("Cannot open pets file"); return; }
    fwrite(&p, sizeof p, 1, fp);
    fclose(fp);
    printf("Pet registered with ID %d.\n", p.id);
}

static void list_pets(void) {
    FILE *fp = fopen(PETS_FILE, "rb");
    if (!fp) { printf("No pets registered yet.\n"); return; }
    Pet p;
    int count = 0;
    printf("\n%-4s %-15s %-10s %-18s %-12s\n", "ID", "Name", "Species", "Owner", "Phone");
    printf("---------------------------------------------------------------\n");
    while (fread(&p, sizeof p, 1, fp) == 1) {
        printf("%-4d %-15s %-10s %-18s %-12s\n", p.id, p.name, p.species, p.owner, p.phone);
        count++;
    }
    fclose(fp);
    if (count == 0) printf("No pets registered yet.\n");
}

static int slot_taken(const char *vet, const char *date, const char *tm) {
    FILE *fp = fopen(APPTS_FILE, "rb");
    if (!fp) return 0;
    Appointment a;
    while (fread(&a, sizeof a, 1, fp) == 1) {
        if (a.status == SCHEDULED && strcmp(a.vet, vet) == 0 &&
            strcmp(a.date, date) == 0 && strcmp(a.time, tm) == 0) {
            fclose(fp);
            return 1;
        }
    }
    fclose(fp);
    return 0;
}

static void book_appointment(void) {
    Appointment a;
    memset(&a, 0, sizeof a);

    int pet_id = read_int("Pet ID: ");
    if (!find_pet(pet_id, NULL)) { printf("No pet with that ID. Register it first.\n"); return; }
    a.pet_id = pet_id;

    read_nonempty("Vet name (e.g. Dr. Rao): ", a.vet, sizeof a.vet);

    char today[11];
    today_string(today);

    do {
        read_line("Date (YYYY-MM-DD): ", a.date, sizeof a.date);
        if (!valid_date(a.date))              printf("  Invalid date format.\n");
        else if (strcmp(a.date, today) < 0) { printf("  Date is in the past.\n"); a.date[0] = '\0'; }
    } while (a.date[0] == '\0' || !valid_date(a.date));

    do {
        read_line("Time (HH:MM, 24-hour): ", a.time, sizeof a.time);
        if (!valid_time(a.time)) printf("  Invalid time format.\n");
    } while (!valid_time(a.time));

    if (slot_taken(a.vet, a.date, a.time)) {
        printf("%s is already booked at %s on %s.\n", a.vet, a.time, a.date);
        return;
    }

    read_line("Reason for visit: ", a.reason, sizeof a.reason);
    a.id = next_id(APPTS_FILE, sizeof(Appointment));
    a.status = SCHEDULED;

    FILE *fp = fopen(APPTS_FILE, "ab");
    if (!fp) { perror("Cannot open appointments file"); return; }
    fwrite(&a, sizeof a, 1, fp);
    fclose(fp);
    printf("Appointment booked with ID %d.\n", a.id);
}

static void list_appointments(void) {
    FILE *fp = fopen(APPTS_FILE, "rb");
    if (!fp) { printf("No appointments yet.\n"); return; }
    Appointment a;
    Pet p;
    int count = 0;
    printf("\n%-4s %-11s %-6s %-12s %-12s %-10s %-10s\n",
           "ID", "Date", "Time", "Pet", "Vet", "Status", "Reason");
    printf("---------------------------------------------------------------------------\n");
    while (fread(&a, sizeof a, 1, fp) == 1) {
        const char *pet_name = find_pet(a.pet_id, &p) ? p.name : "(deleted)";
        printf("%-4d %-11s %-6s %-12s %-12s %-10s %s\n",
               a.id, a.date, a.time, pet_name, a.vet, STATUS_TEXT[a.status], a.reason);
        count++;
    }
    fclose(fp);
    if (count == 0) printf("No appointments yet.\n");
}

static void update_status(int new_status) {
    int id = read_int("Appointment ID: ");
    FILE *fp = fopen(APPTS_FILE, "r+b");
    if (!fp) { printf("No appointments yet.\n"); return; }
    Appointment a;
    while (fread(&a, sizeof a, 1, fp) == 1) {
        if (a.id == id) {
            if (a.status != SCHEDULED) {
                printf("That appointment is already %s.\n", STATUS_TEXT[a.status]);
            } else {
                a.status = new_status;
                fseek(fp, -(long)sizeof a, SEEK_CUR);   /* step back one record */
                fwrite(&a, sizeof a, 1, fp);
                printf("Appointment marked %s.\n", STATUS_TEXT[new_status]);
            }
            fclose(fp);
            return;
        }
    }
    fclose(fp);
    printf("No appointment with that ID.\n");
}

/* ---------- Main menu ---------- */
int main(void) {
    int choice;
    do {
        printf("\n=== Pet Care Clinic ===\n"
               "1. Register pet\n"
               "2. List pets\n"
               "3. Book appointment\n"
               "4. List appointments\n"
               "5. Mark appointment completed\n"
               "6. Cancel appointment\n"
               "0. Exit\n");
        choice = read_int("Choose: ");
        switch (choice) {
            case 1: add_pet(); break;
            case 2: list_pets(); break;
            case 3: book_appointment(); break;
            case 4: list_appointments(); break;
            case 5: update_status(COMPLETED); break;
            case 6: update_status(CANCELLED); break;
            case 0: printf("Goodbye!\n"); break;
            default: printf("Invalid choice.\n");
        }
    } while (choice != 0);
    return 0;
}
