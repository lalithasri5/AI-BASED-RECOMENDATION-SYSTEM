# AI-BASED-RECOMENDATION-SYSTEM
import java.util.*;
import java.util.stream.Collectors;

/**
 * Represents a product with an ID and a name.
 */
class Product {
    private String id;
    private String name;

    public Product(String id, String name) {
        this.id = id;
        this.name = name;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    @Override
    public String toString() {
        return "Product{id='" + id + "', name='" + name + "'}";
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Product product = (Product) o;
        return id.equals(product.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

/**
 * Represents a user with an ID and a map of product IDs to their ratings.
 * A rating of 1.0 indicates a positive preference (e.g., purchased, liked).
 */
class User {
    private String id;
    private Map<String, Double> preferences; // Product ID -> Rating (e.g., 1.0 for liked/purchased)

    public User(String id) {
        this.id = id;
        this.preferences = new HashMap<>();
    }

    public String getId() {
        return id;
    }

    public Map<String, Double> getPreferences() {
        return preferences;
    }

    /**
     * Adds a preference for a product with a given rating.
     * @param productId The ID of the product.
     * @param rating The rating for the product (e.g., 1.0 for liked).
     */
    public void addPreference(String productId, Double rating) {
        preferences.put(productId, rating);
    }

    @Override
    public String toString() {
        return "User{id='" + id + "', preferences=" + preferences + "}";
    }
}

/**
 * A simple recommendation engine that uses user-based collaborative filtering.
 * It finds users similar to the target user and recommends products that similar users liked
 * but the target user has not yet interacted with.
 */
public class RecommendationEngine {

    private Map<String, User> users; // User ID -> User object
    private Map<String, Product> products; // Product ID -> Product object

    public RecommendationEngine() {
        this.users = new HashMap<>();
        this.products = new HashMap<>();
    }

    /**
     * Adds a user to the system.
     * @param user The user object to add.
     */
    public void addUser(User user) {
        users.put(user.getId(), user);
    }

    /**
     * Adds a product to the system.
     * @param product The product object to add.
     */
    public void addProduct(Product product) {
        products.put(product.getId(), product);
    }

    /**
     * Calculates the similarity between two users based on their shared preferences.
     * A higher score indicates more similarity.
     * This simplified version counts the number of shared liked products.
     * In a real system, more sophisticated metrics like Pearson Correlation or Cosine Similarity would be used.
     *
     * @param user1 The first user.
     * @param user2 The second user.
     * @return A similarity score (number of shared liked items).
     */
    private double calculateSimilarity(User user1, User user2) {
        Set<String> user1LikedProducts = user1.getPreferences().keySet();
        Set<String> user2LikedProducts = user2.getPreferences().keySet();

        // Find the intersection of liked products
        Set<String> intersection = new HashSet<>(user1LikedProducts);
        intersection.retainAll(user2LikedProducts);

        // Similarity is the number of common liked products
        return intersection.size();
    }

    /**
     * Finds the most similar users to a given target user.
     *
     * @param targetUserId The ID of the user for whom to find similar users.
     * @param numSimilarUsers The maximum number of similar users to return.
     * @return A list of similar users, sorted by similarity in descending order.
     */
    private List<User> findSimilarUsers(String targetUserId, int numSimilarUsers) {
        User targetUser = users.get(targetUserId);
        if (targetUser == null) {
            System.out.println("Error: Target user " + targetUserId + " not found.");
            return Collections.emptyList();
        }

        Map<User, Double> similarities = new HashMap<>();
        for (User otherUser : users.values()) {
            if (!otherUser.getId().equals(targetUserId)) { // Don't compare user with themselves
                double similarity = calculateSimilarity(targetUser, otherUser);
                if (similarity > 0) { // Only consider users with some shared preferences
                    similarities.put(otherUser, similarity);
                }
            }
        }

        // Sort users by similarity in descending order
        return similarities.entrySet().stream()
                .sorted(Map.Entry.<User, Double>comparingByValue().reversed())
                .limit(numSimilarUsers)
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
    }

    /**
     * Generates product recommendations for a target user.
     * It finds similar users and then suggests products liked by those similar users
     * that the target user has not yet interacted with.
     *
     * @param targetUserId The ID of the user for whom to generate recommendations.
     * @param numSimilarUsers The number of similar users to consider.
     * @param numRecommendations The maximum number of recommendations to return.
     * @return A list of recommended products.
     */
    public List<Product> getRecommendations(String targetUserId, int numSimilarUsers, int numRecommendations) {
        User targetUser = users.get(targetUserId);
        if (targetUser == null) {
            System.out.println("Error: Target user " + targetUserId + " not found.");
            return Collections.emptyList();
        }

        // 1. Find similar users
        List<User> similarUsers = findSimilarUsers(targetUserId, numSimilarUsers);
        if (similarUsers.isEmpty()) {
            System.out.println("No similar users found for " + targetUserId + ". Cannot provide recommendations.");
            return Collections.emptyList();
        }

        // 2. Aggregate preferences from similar users
        Map<String, Double> productScores = new HashMap<>();
        for (User similarUser : similarUsers) {
            for (Map.Entry<String, Double> entry : similarUser.getPreferences().entrySet()) {
                String productId = entry.getKey();
                // Only consider products the target user hasn't interacted with
                if (!targetUser.getPreferences().containsKey(productId)) {
                    // For simplicity, we just count how many similar users liked it.
                    // In a real system, this would be weighted by similarity.
                    productScores.put(productId, productScores.getOrDefault(productId, 0.0) + 1.0);
                }
            }
        }

        // 3. Sort products by score and return top recommendations
        return productScores.entrySet().stream()
                .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                .limit(numRecommendations)
                .map(entry -> products.get(entry.getKey()))
                .filter(Objects::nonNull) // Ensure product exists in our product map
                .collect(Collectors.toList());
    }

    public static void main(String[] args) {
        System.out.println("Starting Recommendation Engine Demo...");

        // Initialize the Recommendation Engine
        RecommendationEngine engine = new RecommendationEngine();

        // --- Sample Products ---
        Product p1 = new Product("P001", "Laptop Pro");
        Product p2 = new Product("P002", "Mechanical Keyboard");
        Product p3 = new Product("P003", "Gaming Mouse");
        Product p4 = new Product("P004", "Noise-Cancelling Headphones");
        Product p5 = new Product("P005", "Smartwatch X");
        Product p6 = new Product("P006", "Ergonomic Chair");
        Product p7 = new Product("P007", "External SSD 1TB");
        Product p8 = new Product("P008", "4K Monitor");

        engine.addProduct(p1);
        engine.addProduct(p2);
        engine.addProduct(p3);
        engine.addProduct(p4);
        engine.addProduct(p5);
        engine.addProduct(p6);
        engine.addProduct(p7);
        engine.addProduct(p8);

        // --- Sample Users and their Preferences (1.0 means liked/purchased) ---

        // User 1: Likes tech gadgets and accessories
        User user1 = new User("User1");
        user1.addPreference("P001", 1.0); // Laptop Pro
        user1.addPreference("P002", 1.0); // Mechanical Keyboard
        user1.addPreference("P003", 1.0); // Gaming Mouse
        user1.addPreference("P007", 1.0); // External SSD
        engine.addUser(user1);

        // User 2: Similar to User1, also likes tech but different items
        User user2 = new User("User2");
        user2.addPreference("P001", 1.0); // Laptop Pro
        user2.addPreference("P004", 1.0); // Noise-Cancelling Headphones
        user2.addPreference("P005", 1.0); // Smartwatch X
        user2.addPreference("P008", 1.0); // 4K Monitor
        engine.addUser(user2);

        // User 3: Likes tech and productivity
        User user3 = new User("User3");
        user3.addPreference("P001", 1.0); // Laptop Pro
        user3.addPreference("P002", 1.0); // Mechanical Keyboard
        user3.addPreference("P006", 1.0); // Ergonomic Chair
        user3.addPreference("P007", 1.0); // External SSD
        engine.addUser(user3);

        // User 4: Likes general electronics
        User user4 = new User("User4");
        user4.addPreference("P004", 1.0); // Noise-Cancelling Headphones
        user4.addPreference("P005", 1.0); // Smartwatch X
        user4.addPreference("P003", 1.0); // Gaming Mouse
        engine.addUser(user4);

        // User 5: New user with some preferences, looking for recommendations
        User user5 = new User("User5");
        user5.addPreference("P001", 1.0); // Laptop Pro
        user5.addPreference("P007", 1.0); // External SSD
        engine.addUser(user5);

        System.out.println("\n--- Current Users and their Preferences ---");
        engine.users.values().forEach(System.out::println);

        System.out.println("\n--- Generating Recommendations for User5 ---");
        List<Product> recommendationsForUser5 = engine.getRecommendations("User5", 3, 5); // Consider 3 similar users, get top 5 recommendations

        if (!recommendationsForUser5.isEmpty()) {
            System.out.println("\nRecommended products for User5:");
            recommendationsForUser5.forEach(product -> System.out.println("- " + product.getName()));
        } else {
            System.out.println("No recommendations found for User5 at this time.");
        }

        System.out.println("\n--- Generating Recommendations for User4 ---");
        List<Product> recommendationsForUser4 = engine.getRecommendations("User4", 2, 3); // Consider 2 similar users, get top 3 recommendations

        if (!recommendationsForUser4.isEmpty()) {
            System.out.println("\nRecommended products for User4:");
            recommendationsForUser4.forEach(product -> System.out.println("- " + product.getName()));
        } else {
            System.out.println("No recommendations found for User4 at this time.");
        }

        System.out.println("\nRecommendation Engine Demo Finished.");
    }
}
