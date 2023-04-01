---
layout: post
title: Plugin architecture with Symfony
---

Symfony [no longer recommends](https://symfony.com/doc/5.4/bundles.html) organising your application using bundles, however bundles are still recommended when you want to share features and functions between multiple applications.

This post examines how bundles are also useful to allow the concept of plugins, or extensions, to be registered dynamically with your core application. It uses the scenario of a [BankingApp](https://github.com/heuristicservices/BankingApp) wanting to support many banking backends, or services.

The idea is that we want to be able to register multiple backends dynamically, using `composer` in the `BankingApp`, in a `BankingService`.

At the end, we want to be able to call these services using a simple key, which in this case is a `string` `acme_bank`:

    $bankService->getBank('acme_bank')->deposit(100);

One of our backends is called "Acme Bank" and is provided by the Symfony bundle [acme-bank-bundle](https://github.com/heuristicservices/acme-bank-bundle).

It provides the `AcmeBankService`:

    class AcmeBankService implements Bank
    {
        public function deposit(int $amount): string
        {
            return sprintf("Deposited %d into AcmeBank", $amount);
        }
    ....

This is registered as a `Symfony` services in `services.php`:

    $services->set(AcmeBankService::class)
        ->public()
        ->autowire()
        ->autoconfigure()
        ->tag('banking_app.bank', []);

Importantly the service is tagged with the key `banking_app.bank`.

We can add the `acme-bank-bundle` to the `BankApp` in the usual way, by installing the bundle using `composer`.

Please be aware, that as these `composer` packages are not part of a public registry, I have added them as local repositories, you will need to update the path if you want to use them:

    "repositories": [
        {
            "type": "path",
            "url": "/home/hs/git/banking-app-api"
        },
        {
            "type": "path",
            "url": "/home/hs/git/acme-bank-bundle"
        }
    ]

You can register the bundle in the usual way in `BankingApp`:

    BankingApp\AcmeBankBundle\BankingAppAcmeBankBundle::class => ['all' => true],

Given the service in `acme-banking-bundle` tagged with `banking_app.bank`, we can add a [compiler pass](https://symfony.com/doc/5.4/service_container/compiler_passes.html) to the `BankingApp` application to detect these services and add it to a registry within our  `BankingService`:

    $bankServiceDefinition = $containerBuilder->findDefinition(BankService::class);
    
    foreach ($containerBuilder->findTaggedServiceIds('banking_app.bank') as $id => $tags) {
        $bankServiceDefinition->addMethodCall('addBank', [new Reference($id)]);
    }

Firstly, this finds our `BankingService`, then when it detects that our bundle has tagged a service with `banking_app.bank`, it will add it to our registry by calling `addBank` on our `BankService`:

    class BankService
    {
        private array $bankRegistry = [];
        public function addBank(Bank $bank): void 
        {
            $this->bankRegistry[$bank->getName()] = $bank;
        }
    }

If you noticed, in our `composer` file, I also included a package called [banking-app-api](https://github.com/heuristicservices/banking-app-api), which has the interface `Bank`:

    interface Bank
    {
        public function deposit(int $amount): string;
        public function getName(): string;
    }

The `BankingApp` and the `banking-app-bundle` both have a dependency on this interface, so that we can use PHP types. This means that the `BankingApp` will never end up calling a method that does not exist on our bank service at runtime.

The API has to exist outside of both projects, as if it were defined in the `BankingApp`, this would introduce a circular dependency, as the `acme-bank-bundle` becomes dependent on the `BankingApp` and the `BankingApp` has a dependency on the `acme-bank-bundle`.

Finally, we can use our `BankService` in the `HomeController` in `BankingApp`:

    #[Route('/')]
    public function home(BankService $bankService) {
        return new Response($bankService->getBank('acme_bank')->deposit(100));
    }

This returns:

     Deposited 100 into AcmeBank

In this case, the `acme_bank` could be a key that is stored in a database, associated with that user. The bank's name comes from a further method we have defined in our API:

     public function getName(): string;

This is a simple example, however all the infrastructure is there to build out more complex systems, for example, you could initiate an OAuth flow using another method in the API:

     public function getOAuthLink(string $userId): string;

You could iterate over your bank registry and display this link for each of the `banking_app.bank` services that have been registered.