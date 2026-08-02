# material-comparison-tool
C++ program for comparing the mass and cost of mechanical parts
# 機械部品の材料・重量・コスト比較ツール
#include <cmath>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <limits>
#include <string>
#include <vector>
#include <filesystem>

class Material {
private:
	std::string name;
	double density;   // g/cm^3
	double pricePerKg; // yen/kg (sample value)

public:
	Material(const std::string& materialName, double materialDensity, double price)
		: name(materialName), density(materialDensity), pricePerKg(price) {}

	const std::string& getName() const { return name; }
	double getDensity() const { return density; }
	double getPricePerKg() const { return pricePerKg; }

	double calculateMassGrams(double volumeCm3) const {
		return density * volumeCm3;
	}

	double calculateCostYen(double massGrams) const {
		return (massGrams / 1000.0) * pricePerKg;
	}
};

struct Result {
	std::string materialName;
	double volumeCm3;
	double massPerPartGrams;
	double totalMassGrams;
	double totalCostYen;
	bool meetsMassLimit;
};

const double PI = 3.141592653589793;

double calculateCylinderVolume(double radiusCm, double heightCm) {
	return PI * radiusCm * radiusCm * heightCm;
}

double calculateRectangularVolume(double widthCm, double depthCm, double heightCm) {
	return widthCm * depthCm * heightCm;
}

double inputPositiveDouble(const std::string& message) {
	double value;
	while (true) {
		std::cout << message;
		if (std::cin >> value && value > 0.0) {
			return value;
		}
		std::cout << "Input error: enter a number greater than 0.\n";
		std::cin.clear();
		std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
	}
}

int inputPositiveInt(const std::string& message) {
	int value;
	while (true) {
		std::cout << message;
		if (std::cin >> value && value > 0) {
			return value;
		}
		std::cout << "Input error: enter an integer greater than 0.\n";
		std::cin.clear();
		std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
	}
}

void saveResultsToCSV(const std::vector<Result>& results, const std::string& filename) {
	std::ofstream fout(filename);
	if (!fout) {
		// 出力ファイルが開けない場合、現在の作業ディレクトリを表示して再試行します。
		try {
			const std::filesystem::path cwd = std::filesystem::current_path();
			std::cerr << "Error: could not open '" << filename << "'.\n";
			std::cerr << "Current working directory: " << cwd.string() << "\n";
			const std::filesystem::path alt = cwd / filename;
			std::cerr << "Attempting to create file at: " << alt.string() << "\n";
			fout.open(alt.string(), std::ios::out);
			if (!fout) {
				std::cerr << "Still could not create file. Check permissions.\n";
				return;
			}
		} catch (const std::exception& ex) {
			std::cerr << "Filesystem error: " << ex.what() << "\n";
			return;
		}
	}

	fout << "Material,Volume_cm3,MassPerPart_g,TotalMass_g,TotalCost_yen,MassLimit\n";
	fout << std::fixed << std::setprecision(2);
	for (const Result& result : results) {
		fout << result.materialName << ','
			 << result.volumeCm3 << ','
			 << result.massPerPartGrams << ','
			 << result.totalMassGrams << ','
			 << result.totalCostYen << ','
			 << (result.meetsMassLimit ? "OK" : "NG") << '\n';
	}
}

int main() {
	// Prices are sample values for program testing, not current market prices.
	const std::vector<Material> materials = {
		Material("Carbon steel", 7.85, 300.0),
		Material("Aluminum", 2.70, 600.0),
		Material("Polyethylene", 0.94, 800.0)
	};

	std::cout << "Mechanical Part Material Comparison Tool\n";
	std::cout << "1: Cylinder\n2: Rectangular prism\n";

	int shape;
	while (true) {
		std::cout << "Select a shape (1 or 2): ";
		if (std::cin >> shape && (shape == 1 || shape == 2)) {
			break;
		}
		std::cout << "Input error: enter 1 or 2.\n";
		std::cin.clear();
		std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
	}

	double volumeCm3 = 0.0;
	if (shape == 1) {
		const double radiusCm = inputPositiveDouble("Radius [cm]: ");
		const double heightCm = inputPositiveDouble("Height [cm]: ");
		volumeCm3 = calculateCylinderVolume(radiusCm, heightCm);
	} else {
		const double widthCm = inputPositiveDouble("Width [cm]: ");
		const double depthCm = inputPositiveDouble("Depth [cm]: ");
		const double heightCm = inputPositiveDouble("Height [cm]: ");
		volumeCm3 = calculateRectangularVolume(widthCm, depthCm, heightCm);
	}

	const int quantity = inputPositiveInt("Quantity: ");
	const double massLimitGrams = inputPositiveDouble("Total mass limit [g]: ");

	std::vector<Result> results;
	results.reserve(materials.size());

	std::cout << "\nComparison results\n";
	std::cout << std::fixed << std::setprecision(2);
	std::cout << std::left << std::setw(16) << "Material"
			  << std::right << std::setw(14) << "Mass/part[g]"
			  << std::setw(15) << "Total mass[g]"
			  << std::setw(14) << "Cost[yen]"
			  << std::setw(9) << "Limit" << '\n';

	for (const Material& material : materials) {
		const double massPerPart = material.calculateMassGrams(volumeCm3);
		const double totalMass = massPerPart * quantity;
		const double totalCost = material.calculateCostYen(totalMass);
		const bool meetsLimit = totalMass <= massLimitGrams;

		results.push_back({material.getName(), volumeCm3, massPerPart,
						   totalMass, totalCost, meetsLimit});

		std::cout << std::left << std::setw(16) << material.getName()
				  << std::right << std::setw(14) << massPerPart
				  << std::setw(15) << totalMass
				  << std::setw(14) << totalCost
				  << std::setw(9) << (meetsLimit ? "OK" : "NG") << '\n';
	}

	const Result* lightest = &results.front();
	const Result* cheapest = &results.front();
	for (const Result& result : results) {
		if (result.totalMassGrams < lightest->totalMassGrams) {
			lightest = &result;
		}
		if (result.totalCostYen < cheapest->totalCostYen) {
			cheapest = &result;
		}
	}

	std::cout << "\nLightest material: " << lightest->materialName << '\n';
	std::cout << "Lowest estimated cost: " << cheapest->materialName << '\n';

	saveResultsToCSV(results, "output.csv");
	std::cout << "Results were saved to output.csv.\n";
	return 0;
}
