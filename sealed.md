**What is a Sealed Type?**

From Java 17 onwards, a class or interface can explicitly specify its direct subtypes, meaning it can control which classes or interfaces are allowed to extend or implement it.

Before Java 17, a class or interface had no control over its subtypes. Any class or interface could extend or implement it unless explicitly marked final. So, we had two extremes: either a type was open to all (public interface/class) for extension, or it was completely closed (final class). Sealed Types introduce a middle ground, allowing fine-grained control over the inheritance hierarchy.

Many projects deal with well-defined domains where the type hierarchy is known in advance. For example, in our previous lectures, we worked on payment processing functionality. The domain is well-defined, and as developers, we know that the Payment interface will only have four subtypes: CreditCard, GooglePay, PayPal, and MembershipType. By enforcing this restriction (domain knowledge) in the interface declaration, we make our code more clearer, readable and maintainable. This is precisely what Sealed Types enable. We can define our Payment interface as a sealed interface as follows:

sealed interface Payment permits CreditCard, GooglePay, PayPal, MembershipType {}

Here, we use the sealed keyword to declare Payment as a sealed interface, followed by the permits keyword to specify its allowed direct subtypes. If any other class or interface tries to extend or implement Payment, it would see a compilation error. Below the entire code is given. You can in fact change just the above declaration of Payment and everything works as earlier.

**Benefits of Sealed Types:**

1. Ease of Reasoning

As stated above, someone looking at the above declaration now clearly understands that Payment can have exactly 4 direct subtypes.

2. Exhaustivity in Patter Switch

Sealed Types enable safer pattern matching in switch. Since the compiler knows all possible subtypes, we no longer need a default case. The compiler ensures that every permitted subtype is handled in a case block. If we omit a case for any of the permitted subtypes, we get a compilation error.

If a new subtype is added to the permits list in the future, the compiler forces us to update the switch accordingly, preventing unintended runtime issues.

Constraints on Permitted Subtypes:

1. Every permitted subtype must directly extend the sealed type. Otherwise, we get a compiler error.

2. Sealed type and its permitted subtypes must belong to the same package (or the same module if using Java Modules from Java 9). This enforces a close relationship between them as being part of the same domain.

3. A permitted subtype can be declared as final or sealed or non-sealed. The keywords' sealed or non-sealed control how the sealing property propagates. If a subtype is declared as sealed, it just means it permits only certain subtypes. Similarly, if a subtype is declared with non-sealed keyword, then it is open for extension by any subtype. Declaring as non-sealed is acceptable because the supertype has already verified it as a trusted subtype.

PaymentProcessor Code:

Below is PaymentProcessor code with Payment as sealed interface. The permitted subtypes in this case are record classes and an enum. We know that a record class and enum and implicitly converted into a final class and hence they meet constraint of being a final subtype.


```
package com.semanticsquare.modernjava.patternmatching;
 
sealed interface Payment permits CreditCard, GooglePay, PayPal, MembershipType {}
 
record CreditCard (String cardNum, String cvv) implements Payment {}
record GooglePay (String upid) implements Payment {}
record PayPal (String email) implements Payment {}
 
enum MembershipType implements Payment { GOLD, PLATINUM }
 
public class PaymentProcessor {	
	
	public String processPayment(Payment payment, double amount) {
			return switch (payment) {	
				
				case CreditCard cc when !isValidCard(cc.cvv()) -> "Invalid CreditCard";
				case CreditCard cc when amount <= 10.0 -> "Amount too low";
				case CreditCard cc -> {	
					processCC(cc.cardNum(), cc.cvv(), amount);
					yield "CreditCard";
				}
				
				
				case GooglePay gp -> {
					processGPay(gp.upid(), amount);
					yield "GooglePay";
				}
				case PayPal pp -> {
					processPPal(pp.email(), amount);
					yield "PayPal";
				}
				
				case MembershipType.PLATINUM -> "PlatinumFree";
				case MembershipType mt -> "Gold50OFF";				
			};
		
	}
	
	private boolean isValidCard(String cvv) {
		return cvv.equals("122");
	}
	
	private void processCC(String cardNum, String cvv, double amount) {
		System.out.println("Processing Credit Card with card number: " + cardNum + " for amount " + amount);
	}
 
	private void processGPay(String upid, double amount) {
		System.out.println("Processing Google Pay with UPI ID: " + upid + " for amount " + amount);
	}
 
	private void processPPal(String email, double amount) {
		System.out.println("Processing PayPal with email: " + email + " for amount " + amount);
	}
	
	public boolean applyCoupon(String code) {
		return switch (code) {
			case "NEWYEAR25", "XMAS25" -> true;
			case String s when s.equalsIgnoreCase("DIWALI123")  -> true;
			case String s -> false;
		};
	}
 
	public static void main(String[] args) {
		PaymentProcessor processor = new PaymentProcessor();
 
		Payment cc = new CreditCard("1234-5678-9012-3456", "122");
		
		Payment gp = new GooglePay("user@upi");
		Payment pp = new PayPal("user@example.com");
		
		System.out.println("Payment Type: " + processor.processPayment(cc, 200.0) + "\n"); 
		
		System.out.println("Payment Type: " + processor.processPayment(gp, 200.0) + "\n"); 
		System.out.println("Payment Type: " + processor.processPayment(pp, 200.0) + "\n"); 
		
		System.out.println("Payment Type: " + processor.processPayment(MembershipType.PLATINUM, 200.0) + "\n");
		
		System.out.println("is NEWYEAR25 valid? " + processor.applyCoupon("NEWYEAR25"));
		System.out.println("is DIWALI123 valid? " + processor.applyCoupon("diwali123"));
	}
}

```
