# WpfApp

📁 WpfApp: Directory and File Server
WpfApp은 Winsock2를 사용하여 C로 작성된 간단한 디렉토리 및 파일 서버입니다. 이 서버는 클라이언트의 요청을 수신하여 디렉토리 목록을 반환하거나 특정 명령에 따라 파일 관련 응답을 제공합니다.

🛠️ 주요 기능
디렉토리 목록 조회: 지정된 디렉토리의 파일과 폴더를 나열합니다.
명령 기반 처리: GET_DIRECTORY 및 GET_FILES와 같은 명령을 지원합니다.
경량 및 효율적: Winsock과 Windows API를 사용하여 높은 성능을 제공합니다.
🖥️ 작동 방식
서버는 클라이언트로부터 명령을 수신하며, 아래 명령들을 처리합니다.

지원되는 명령
GET_DIRECTORY|PATH=<directory_path>
지정된 경로의 디렉토리 내용을 반환합니다.

GET_FILES
파일 관련 요청을 처리하기 위한 예약된 명령입니다. 현재는 단순히 확인 응답을 보냅니다.

기타 명령
알 수 없는 명령이 들어오면 Unknown command 메시지를 반환합니다.


agent.c   
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <winsock2.h>
#include <windows.h> 

#pragma comment(lib, "Ws2_32.lib")

#define PORT 8080
#define BUFFER_SIZE 1024

void handle_directory_request(const char *directory_path, SOCKET client_socket) {
	WIN32_FIND_DATA findFileData;
	HANDLE hFind;

	char search_path[BUFFER_SIZE];
	snprintf(search_path, sizeof(search_path), "%s\\*", directory_path);

	hFind = FindFirstFile(search_path, &findFileData);
	if (hFind == INVALID_HANDLE_VALUE) {
		const char *error_response = "Invalid directory path or access denied.";
		send(client_socket, error_response, strlen(error_response), 0);
		return;
	}

	char response[BUFFER_SIZE] = "Directory contents:\n";
	do {
		strcat_s(response, BUFFER_SIZE, findFileData.cFileName);
		strcat_s(response, BUFFER_SIZE, "\n");
	} while (FindNextFile(hFind, &findFileData) != 0);

	FindClose(hFind);
	send(client_socket, response, strlen(response), 0);
}

void handle_client(SOCKET client_socket) {
	char buffer[BUFFER_SIZE];
	int bytes_received = recv(client_socket, buffer, BUFFER_SIZE, 0);

	if (bytes_received == SOCKET_ERROR) {
		printf("Error receiving data: %d\n", WSAGetLastError());
		closesocket(client_socket);
		return;
	}

	buffer[bytes_received] = '\0';
	printf("Received data: %s\n", buffer);

	if (strstr(buffer, "GET_DIRECTORY")) {
		char directory_path[BUFFER_SIZE];
		sscanf_s(buffer, "GET_DIRECTORY|PATH=%s", directory_path, (unsigned)_countof(directory_path));

		handle_directory_request(directory_path, client_socket);
	}
	else if (strstr(buffer, "GET_FILES")) {
		const char *response = "Server acknowledges file request";
		send(client_socket, response, strlen(response), 0);
	}
	else {
		const char *response = "Unknown command";
		send(client_socket, response, strlen(response), 0);
	}

	closesocket(client_socket);
}

int main() {
	WSADATA wsaData;
	SOCKET server_socket, client_socket;
	struct sockaddr_in server_addr, client_addr;
	int addr_len = sizeof(client_addr);

	if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
		printf("WSAStartup failed: %d\n", WSAGetLastError());
		return EXIT_FAILURE;
	}

	server_socket = socket(AF_INET, SOCK_STREAM, 0);
	if (server_socket == INVALID_SOCKET) {
		printf("Socket creation failed: %d\n", WSAGetLastError());
		WSACleanup();
		return EXIT_FAILURE;
	}

	server_addr.sin_family = AF_INET;
	server_addr.sin_addr.s_addr = INADDR_ANY;
	server_addr.sin_port = htons(PORT);

	if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) == SOCKET_ERROR) {
		printf("Bind failed: %d\n", WSAGetLastError());
		closesocket(server_socket);
		WSACleanup();
		return EXIT_FAILURE;
	}

	if (listen(server_socket, 5) == SOCKET_ERROR) {
		printf("Listen failed: %d\n", WSAGetLastError());
		closesocket(server_socket);
		WSACleanup();
		return EXIT_FAILURE;
	}

	printf("Server listening on port %d...\n", PORT);

	while (1) {
		client_socket = accept(server_socket, (struct sockaddr *)&client_addr, &addr_len);
		if (client_socket == INVALID_SOCKET) {
			printf("Accept failed: %d\n", WSAGetLastError());
			continue;
		}

		handle_client(client_socket);
	}

	closesocket(server_socket);
	WSACleanup();
	return 0;
}
