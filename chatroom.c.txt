#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define MAX_MSG_LEN 512
#define MAX_NUM_CHATBOT 10

int fd[MAX_NUM_CHATBOT+1][2];  // Global pipes array

void panic(char *s) {
    fprintf(2, "%s\n", s);
    exit(1);
}

int fork1(void) {
    int pid = fork();
    if (pid == -1) 
        panic("fork");
    return pid;
}

void pipe1(int fd[2]) {
    if (pipe(fd) < 0) 
        panic("Fail to create a pipe.");
}

void gets1(char msgBuf[MAX_MSG_LEN]) {
    gets(msgBuf, MAX_MSG_LEN);
    int len = strlen(msgBuf);
    if (len > 0) 
        msgBuf[len - 1] = '\0';
}

void chatbot(int myId, char *myName, char *botNames[], int numBots) {
    int i;
    char recvMsg[MAX_MSG_LEN], msgBuf[MAX_MSG_LEN], newBotName[MAX_MSG_LEN];

    // Close unused pipes
    for (i = 0; i < myId - 1; i++) {
        close(fd[i][0]);
        close(fd[i][1]);
    }
    close(fd[myId - 1][1]); // Close previous bot's write pipe
    close(fd[myId][0]);     // Close my read pipe

    while (1) {
        memset(recvMsg, 0, MAX_MSG_LEN);
        // Wait for activation message
        read(fd[myId - 1][0], recvMsg, MAX_MSG_LEN);

        // Handle EXIT command
        if (strcmp(recvMsg, ":EXIT") == 0 || strcmp(recvMsg, ":exit") == 0) {
            write(fd[myId][1], recvMsg, MAX_MSG_LEN);
            exit(0);
        }

        // Check if this bot should be active
        if (strcmp(recvMsg, myName) == 0 || strcmp(recvMsg, ":START") == 0) {
            printf("Hello, this is chatbot %s. Type ':CHANGE' to switch bots or ':EXIT' to quit.\n", myName);

            while (1) {
                memset(msgBuf, 0, MAX_MSG_LEN);
                gets1(msgBuf);

                if (strcmp(msgBuf, ":EXIT") == 0 || strcmp(msgBuf, ":exit") == 0) {
                    write(fd[myId][1], msgBuf, MAX_MSG_LEN);
                    exit(0);
                }

                if (strcmp(msgBuf, ":CHANGE") == 0 || strcmp(msgBuf, ":change") == 0) {
                    printf("Enter the name of the bot you want to switch to: ");
                    memset(newBotName, 0, MAX_MSG_LEN);
                    gets1(newBotName);

                    int found = 0;
                    for (i = 1; i < numBots; i++) {
                        if (strcmp(newBotName, botNames[i]) == 0) {
                            found = 1;
                            break;
                        }
                    }

                    if (found) {
                        if (strcmp(newBotName, myName) == 0) {
                            printf("Already chatting with %s!\n", myName);
                        } else {
                            printf("Switching to %s...\n", newBotName);
                            write(fd[myId][1], newBotName, MAX_MSG_LEN);
                            break;
                        }
                    } else {
                        printf("Bot not found! Try again.\n");
                    }
                } else {
                    printf("%s: I heard you say: %s\n", myName, msgBuf);
                }
            }
        } else {
            // Pass message to next bot if not for me
            write(fd[myId][1], recvMsg, MAX_MSG_LEN);
        }
    }
}

int main(int argc, char *argv[]) {
    int i;

    if (argc < 3 || argc > MAX_NUM_CHATBOT + 1) {
        printf("Usage: %s <list of names for up to %d chatbots>\n", argv[0], MAX_NUM_CHATBOT);
        exit(1);
    }

    // Create pipes
    pipe1(fd[0]);
    for (i = 1; i < argc; i++) {
        pipe1(fd[i]);
        if (fork1() == 0) {
            chatbot(i, argv[i], argv, argc);
        }
    }

    // Close unused pipes in parent
    close(fd[0][0]);
    close(fd[argc - 1][1]);
    for (i = 1; i < argc - 1; i++) {
        close(fd[i][0]);
        close(fd[i][1]);
    }

    // Start chat with first bot
    write(fd[0][1], ":START", MAX_MSG_LEN);

    // Forward messages
    while (1) {
        char recvMsg[MAX_MSG_LEN];
        memset(recvMsg, 0, MAX_MSG_LEN);
        read(fd[argc - 1][0], recvMsg, MAX_MSG_LEN);
        
        if (strcmp(recvMsg, ":EXIT") == 0 || strcmp(recvMsg, ":exit") == 0) {
            write(fd[0][1], recvMsg, MAX_MSG_LEN);
            break;
        }
        
        write(fd[0][1], recvMsg, MAX_MSG_LEN);
    }

    // Wait for all child processes
    for (i = 1; i < argc; i++) 
        wait(0);

    printf("Now the chatroom closes. Bye bye!\n");
    exit(0);
}