1. try-with-resource
		try (Scanner scanner = new Scanner(new File("test.txt"))) { 
		while (scanner.hasNext()) { System.out.println(scanner.nextLine()); } 
		} catch (FileNotFoundException fnfe) { fnfe.printStackTrace(); 
	}