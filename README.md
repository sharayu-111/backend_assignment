<?php
// src/Controller/UserController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;
use Doctrine\ORM\EntityManagerInterface;
use App\Entity\User;
use App\Service\CsvService;
use App\Service\EmailService;
use App\Service\TwitterAuthService;

class UserController extends AbstractController
{
    private $entityManager;
    private $csvService;
    private $emailService;
    private $twitterAuthService;

    public function __construct(EntityManagerInterface $entityManager, CsvService $csvService, EmailService $emailService, TwitterAuthService $twitterAuthService)
    {
        $this->entityManager = $entityManager;
        $this->csvService = $csvService;
        $this->emailService = $emailService;
        $this->twitterAuthService = $twitterAuthService;
    }

    #[Route('/api/upload', name: 'upload_csv', methods: ['POST'])]
    public function upload(Request $request): Response
    {
        $file = $request->files->get('file');
        if (!$file) {
            return $this->json(['error' => 'No file uploaded'], Response::HTTP_BAD_REQUEST);
        }

        $users = $this->csvService->parseCsv($file);
        foreach ($users as $userData) {
            $user = new User();
            $user->setName($userData['name']);
            $user->setEmail($userData['email']);
            $user->setUsername($userData['username']);
            $user->setAddress($userData['address']);
            $user->setRole($userData['role']);
            
            $this->entityManager->persist($user);
            $this->emailService->sendEmail($userData['email'], 'Welcome!', 'Your data has been successfully stored.');
        }
        $this->entityManager->flush();

        return $this->json(['message' => 'File uploaded and users stored successfully']);
    }

    #[Route('/api/users', name: 'get_users', methods: ['GET'])]
    public function getUsers(): Response
    {
        $users = $this->entityManager->getRepository(User::class)->findAll();
        return $this->json($users);
    }

    #[Route('/api/backup', name: 'backup_database', methods: ['GET'])]
    public function backupDatabase(): Response
    {
        $backupFile = $this->csvService->backupDatabase();
        return $this->json(['message' => 'Database backup created', 'file' => $backupFile]);
    }

    #[Route('/api/restore', name: 'restore_database', methods: ['POST'])]
    public function restoreDatabase(Request $request): Response
    {
        $file = $request->files->get('backup');
        if (!$file) {
            return $this->json(['error' => 'No backup file uploaded'], Response::HTTP_BAD_REQUEST);
        }
        $this->csvService->restoreDatabase($file);
        return $this->json(['message' => 'Database restored successfully']);
    }

    #[Route('/auth/twitter', name: 'twitter_auth', methods: ['GET'])]
    public function twitterAuth(): Response
    {
        $authUrl = $this->twitterAuthService->getAuthUrl();
        return $this->redirect($authUrl);
    }

    #[Route('/auth/twitter/callback', name: 'twitter_callback', methods: ['GET'])]
    public function twitterCallback(Request $request): Response
    {
        $userData = $this->twitterAuthService->handleCallback($request);
        
        $user = new User();
        $user->setName($userData['name']);
        $user->setEmail($userData['email']);
        $user->setUsername($userData['username']);
        
        $this->entityManager->persist($user);
        $this->entityManager->flush();

        return $this->json(['message' => 'Twitter authentication successful', 'user' => $userData]);
    }
}
