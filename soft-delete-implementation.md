# Complete Implementation Guide: Soft Delete in Symfony with API Platform

This guide provides all the code and steps needed to implement soft delete functionality in a Symfony application with API Platform.

## Directory Structure
```
src/
├── Entity/
│   └── YourEntity.php
├── Trait/
│   └── SoftDeleteTrait.php
├── Repository/
│   └── YourEntityRepository.php
├── Filter/
│   └── ShowDeletedFilter.php
├── State/
│   └── SoftDeleteProcessor.php
└── Doctrine/
    └── Filter/
        └── SoftDeleteFilter.php
```

## 1. Create the Soft Delete Trait
File: `src/Trait/SoftDeleteTrait.php`
```php
<?php

namespace App\Trait;

use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

trait SoftDeleteTrait
{
    #[ORM\Column(type: Types::SMALLINT, options: ["default" => 0])]
    private ?int $is_deleted = 0; //? 0 = not deleted, 1 = deleted

    public function getIsDeleted(): ?int
    {
        return $this->is_deleted;
    }

    public function setIsDeleted(int $is_deleted): static
    {
        $this->is_deleted = $is_deleted;
        return $this;
    }

    public function softDelete(): static
    {
        $this->is_deleted = 1;
        return $this;
    }

    public function restore(): static
    {
        $this->is_deleted = 0;
        return $this;
    }
}
```

## 2. Create the Entity
File: `src/Entity/YourEntity.php`
```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Delete;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;
use Doctrine\ORM\Mapping as ORM;
use App\Repository\YourEntityRepository;
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Serializer\Annotation\Groups;
use App\State\SoftDeleteProcessor;
use ApiPlatform\Metadata\ApiFilter;
use App\Filter\ShowDeletedFilter;

#[ORM\Entity(repositoryClass: YourEntityRepository::class)]
#[ORM\HasLifecycleCallbacks]
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection(),
        new Post(),
        new Put(),
        new Delete(
            processor: SoftDeleteProcessor::class
        )
    ],
    normalizationContext: ['groups' => ['read']],
    denormalizationContext: ['groups' => ['write']]
)]
#[ApiFilter(ShowDeletedFilter::class)]
class YourEntity
{
    use \App\Trait\SoftDeleteTrait;

    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    #[Groups(['read'])]
    private ?int $id = null;

    // Add your other entity properties here
    
    public function getId(): ?int
    {
        return $this->id;
    }

    // Add your other getters and setters here
}
```

## 3. Create the Repository
File: `src/Repository/YourEntityRepository.php`
```php
<?php

namespace App\Repository;

use App\Entity\YourEntity;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class YourEntityRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, YourEntity::class);
    }

    public function findAllIncludingDeleted()
    {
        return $this->createQueryBuilder('e')
            ->getQuery()
            ->getResult();
    }

    public function findAllActive()
    {
        return $this->createQueryBuilder('e')
            ->andWhere('e.is_deleted = 0')
            ->getQuery()
            ->getResult();
    }
}
```

## 4. Create the Soft Delete Filter
File: `src/Doctrine/Filter/SoftDeleteFilter.php`
```php
<?php

namespace App\Doctrine\Filter;

use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

class SoftDeleteFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        if (!$targetEntity->hasField('is_deleted')) {
            return '';
        }

        return sprintf('%s.is_deleted = 0', $targetTableAlias);
    }
}
```

## 5. Create the Show Deleted Filter
File: `src/Filter/ShowDeletedFilter.php`
```php
<?php

namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\AbstractFilter;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;

final class ShowDeletedFilter extends AbstractFilter
{
    protected function filterProperty(
        string $property,
        $value,
        QueryBuilder $queryBuilder,
        QueryNameGeneratorInterface $queryNameGenerator,
        string $resourceClass,
        Operation $operation = null,
        array $context = []
    ): void {
        if ($property !== 'showDeleted') {
            return;
        }

        $rootAlias = $queryBuilder->getRootAliases()[0];
        
        if ($value === true || $value === 'true') {
            $em = $queryBuilder->getEntityManager();
            $em->getFilters()->disable('softdelete');
        }
    }

    public function getDescription(string $resourceClass): array
    {
        return [
            'showDeleted' => [
                'property' => 'showDeleted',
                'type' => 'boolean',
                'required' => false,
                'swagger' => [
                    'description' => 'Filter to show soft deleted records',
                    'name' => 'showDeleted',
                    'type' => 'boolean',
                ],
            ],
        ];
    }
}
```

## 6. Create the Soft Delete Processor
File: `src/State/SoftDeleteProcessor.php`
```php
<?php

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use Doctrine\ORM\EntityManagerInterface;

class SoftDeleteProcessor implements ProcessorInterface
{
    public function __construct(
        private EntityManagerInterface $entityManager
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): void
    {
        if (method_exists($data, 'softDelete')) {
            $data->softDelete();
            $this->entityManager->persist($data);
            $this->entityManager->flush();
        }
    }
}
```

## 7. Configuration Files

### services.yaml
```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'

    App\State\SoftDeleteProcessor:
        arguments:
            $entityManager: '@doctrine.orm.entity_manager'
        tags: ['api_platform.state_processor']

    App\Filter\ShowDeletedFilter:
        tags: ['api_platform.filter']
```

### doctrine.yaml
```yaml
doctrine:
    orm:
        filters:
            softdelete:
                class: App\Doctrine\Filter\SoftDeleteFilter
                enabled: true
```

## 8. Usage Examples

### Try via post man

### API Usage
```bash
# Soft delete an entity
DELETE /api/your_entities/1

# Get all entities including deleted ones
GET /api/your_entities?showDeleted=true

# Get only active entities
GET /api/your_entities
```

This implementation provides a complete soft delete solution that integrates with Symfony's entity management system and API Platform, while maintaining data integrity and providing flexible querying options.
