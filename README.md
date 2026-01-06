# Process-Radio
Process Radio is a C-based OS project that simulates radio broadcasting using fork, pipes, threads, mutexes, and signals. It demonstrates inter-process communication, multithreading, non-blocking input handling, and graceful process control on Linux systems.listener.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>
#include <signal.h>
#include <string.h>

#define MAX_SONGS 100
#define LINE_LEN  256

    char song_buffer[MAX_SONGS][LINE_LEN];
    int count=0;
    int running =1;
       pthread_mutex_t lock;

    void handle_sigterm(int sig){

    running=0;
    printf("\n Listener shutting down...\n");
   }


    void *receiver_thread(void *arg) {
    FILE *in = fdopen(STDIN_FILENO,"r");
    if(!in){
        perror("fdopen");
        return NULL;
    }

    char line[LINE_LEN];

    while(running&&fgets(line, sizeof(line),in)){
        // newline hatao
        size_t n=strlen(line);
        if (n>0 && line[n-1] =='\n')line[n-1] = '\0';

        pthread_mutex_lock(&lock);
        if(count<MAX_SONGS){
            strncpy(song_buffer[count],line,LINE_LEN-1);
            song_buffer[count][LINE_LEN-1] = '\0';
            count++;
        }
       pthread_mutex_unlock(&lock);
       printf("Received: %s\n",line);
       sleep(1); 
    }
       return NULL;
}

     void *logger_thread(void *arg){
    int id = *(int *)arg;
    char filename[64];
    sprintf(filename,"listener%d_log.txt",id);

    FILE *log=fopen(filename,"a");
    if (!log) {
        perror("Error opening log file");
        return NULL;
    }

    while(running){
        pthread_mutex_lock(&lock);
        for(int i=0;i<count;i++) {
            fprintf(log,"%s\n",song_buffer[i]);
        }
        count=0;
        pthread_mutex_unlock(&lock);

        fflush(log);
        sleep(3);   
    }

    // exit se pehle bacha hua data bhi likh do
    pthread_mutex_lock(&lock);
    for(int i=0;i<count;i++){
        fprintf(log,"%s\n",song_buffer[i]);
    }
    pthread_mutex_unlock(&lock);
    fflush(log);

    fclose(log);
    return NULL;
}

     int main(int argc,char *argv[]){
    if(argc<2){
        fprintf(stderr,"Usage:listener <id>\n");
        return 1;
    }
    int id=atoi(argv[1]);

    signal(SIGTERM,handle_sigterm);
    signal(SIGINT,handle_sigterm);

    printf("Listener %d starting (2 threads: receiver + logger)\n",id);

    pthread_mutex_init(&lock,NULL);

    pthread_t t_recv, t_log;
    pthread_create(&t_recv,NULL,receiver_thread,NULL);
    pthread_create(&t_log,NULL,logger_thread,&id);

    pthread_join(t_recv,NULL);
    running=0;  //ensure logger loop exits
    pthread_join(t_log,NULL);

    pthread_mutex_destroy(&lock);

    printf("Listener %d exited.\n",id);
    return 0;
}



// radio_station.c

#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/select.h>   
#include <sys/time.h>    
#include <sys/types.h>  
#include <errno.h>

     pid_t pid1 = 0, pid2 = 0;
    void show_menu(){
    printf("\n --- Controls ---\n");
    printf("p : Pause listeners\n");
    printf("r : Resume listeners\n");
    printf("s : Stop broadcast\n");
     printf("------------------\n");
    }
      int main(){
    int fd1[2],fd2[2];
    if (pipe(fd1)==-1)
    {perror("pipe1"); return 1;}
    if (pipe(fd2)==-1){perror("pipe2"); return 1;}

    pid1=fork();
    if (pid1 < 0) { perror("fork1"); return 1; }
    if (pid1==0) {
        //child1
        close(fd1[1]);
        close(fd2[0]); close(fd2[1]);
        dup2(fd1[0], STDIN_FILENO);
        execl("./listener", "listener", "1", NULL);
        perror("execl listener1");
        _exit(1);
    }

    pid2 = fork();
    if (pid2 < 0) { perror("fork2"); return 1; }
    if (pid2 == 0) {
        // child2
        close(fd2[1]);
        close(fd1[0]); close(fd1[1]);
        dup2(fd2[0], STDIN_FILENO);
        execl("./listener", "listener", "2", NULL);
        perror("execl listener2");
        _exit(1);
    }
    close(fd1[0]);
    close(fd2[0]);

    printf("\n Multithreaded Process Radio Started!\n");
    printf("Listener1 pid=%d  Listener2 pid=%d\n", pid1, pid2);
    show_menu();

    char *songs[] = {"Why this kolaveri di", " Mai hu gian", "All is well", "Sunshine","kuch kuch hota hai","love story","dance monkey"};
    int total=sizeof(songs)/sizeof(songs[0]);
    int paused = 0;
    char cmdline[64];

    while(1){    
    if(!paused){
    char *song=songs[rand()%total];
    printf("\nBroadcasting: %s\n",song);
    ssize_t w;

    w= write(fd1[1],song,strlen(song));
    if(w==-1 && errno != EPIPE) perror("write fd1");
    write(fd1[1],"\n",1);

    w =write(fd2[1], song, strlen(song));
    if(w==-1 && errno!=EPIPE) perror("write fd2");
    write(fd2[1], "\n", 1);
    }else{
    printf("\n(Broadcast paused — not sending new songs)\n");
    }

        for (int t=0;t<4;++t){
            fd_set rfds;
            FD_ZERO(&rfds);
            FD_SET(STDIN_FILENO, &rfds);
            struct timeval tv = {.tv_sec =0, .tv_usec=500000}; //0.5s

            int sel=select(STDIN_FILENO + 1, &rfds, NULL, NULL, &tv);
            if (sel>0 && FD_ISSET(STDIN_FILENO, &rfds)) {
                if(fgets(cmdline, sizeof(cmdline), stdin) == NULL) {
                    continue;
                }
                char cmd = cmdline[0];
                if(cmd=='p') {
                    if (pid1) kill(pid1, SIGSTOP);
                    if (pid2) kill(pid2, SIGSTOP);
                    paused = 1;
                    printf("Paused listeners.\n");
                } else if (cmd == 'r') {
                    if (pid1) kill(pid1, SIGCONT);
                    if (pid2) kill(pid2, SIGCONT);
                    paused = 0;
                    printf("Resumed listeners.\n");
                } else if (cmd == 's') {
                    printf("Stopping broadcast...\n");
                    if (pid1) kill(pid1, SIGTERM);
                    if (pid2) kill(pid2, SIGTERM);
                    // close write ends so listeners get EOF
                    close(fd1[1]); close(fd2[1]);
                    goto wait_children;
                } else {
                    show_menu();
                }
                show_menu();
            }
        }
    }

     wait_children:
    if (pid1) waitpid(pid1, NULL, 0);
    if (pid2) waitpid(pid2, NULL, 0);
    printf(" Broadcast ended.\n");
    return 0;
    }
